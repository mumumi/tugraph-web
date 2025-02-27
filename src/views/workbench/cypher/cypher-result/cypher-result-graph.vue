<template>
    <div class="workbench-cypher-result-graph" ref="workbenchCypherResultGraph">
        <div class="result-graph-canvas" ref="resultGraphCanvas"></div>
        <div class="lalala" v-show="false">
            <Cxttapend ref="cxttapend" @closeCxttapend="closeCxttapend" :tabValue="tabValue" :targetVertexID="targetVertexID" />
        </div>
    </div>
</template>
<script lang="ts">
import store from '@/store/index'
import { Component, Prop, Vue, Watch } from 'vue-property-decorator'
import { getModule } from 'vuex-module-decorators'
import { CYTOSCAPECONFIG as CyConfig, ADDEDGEOPTION } from './cytoscape-config'
import CypherStore from '@/store/cypher/cypher'
import SubGraphManageStore from '@/store/subgraph-manage/subgraph-manage'
import { downloadFile } from '@/core/uitls'

import cytoscape from 'cytoscape'
import euler from 'cytoscape-euler'
import dblclick from 'cytoscape-dblclick'
import popper from 'cytoscape-popper'
import { forceOption, gridOption, treeOption, circleOption } from './cypher-result-graph-layout'
import edgehandles from 'cytoscape-edgehandles'

import Cxttapend from './cypher-result-graph-cxttapend.vue'

cytoscape.use(edgehandles)
cytoscape.use(dblclick)
cytoscape.use(euler)
cytoscape.use(popper)

interface cyData {
    data: {
        [propName: string]: any
    }
    classes: Array<string>
    [propName: string]: any
}
@Component({ components: { Cxttapend } })
export default class WorkbenchCypherResultGraph extends Vue {
    public $refs!: {
        resultGraphCanvas: HTMLElement
        workbenchCypherResultGraph: HTMLElement
        cxttapend: Vue
    }
    targetVertexID: number = 0
    cypherStore: CypherStore = getModule(CypherStore, store)
    subGraphManageStore: SubGraphManageStore = getModule(SubGraphManageStore, store)
    _cy: any = null
    _eh: any = null
    _layout: any = null
    fixNodeList: { [propName: string]: any } = {}
    _currenDbClickNode: any = null
    _cyDom: any = null
    // 初始缩放等级
    zoomLevel: number = 1
    cyNodes: Array<any> = []
    cyEdges: Array<any> = []
    currentLayoutOption: any = forceOption
    forceOption: any = forceOption
    gridOption: any = gridOption
    treeOption: any = treeOption
    circleOption: any = circleOption
    // addEdgeStatus是true开始添加边，可以进行连线的开关
    addEdgeStatus: boolean = false

    @Prop(String) tabValue!: string
    get activeTabValue() {
        return this.cypherStore.tabValue
    }
    @Watch('activeTabValue')
    onChangeActiveTabValue() {
        this.fixed()
        if (this.activeTabValue === this.tabValue) {
            this.unfixed()
            if (this.currentLayoutOption.name === 'euler') {
                this.panSource()
            }
        }
    }
    get fixedActive() {
        let data = this.cypherStore.cypherReasultDatas.find((item) => item.id === this.tabValue)
        let status = data ? data.btns.fixed.active : false
        return status
    }
    get isMergeEdge() {
        let data = this.cypherStore.cypherReasultDatas.find((item) => item.id === this.tabValue)
        return data && data.btns.mergeEdge.active
    }
    @Watch('isMergeEdge')
    onChangeIsMergeEdge() {
        this.mergeEdge()
    }
    get isHover() {
        let data = this.cypherStore.cypherReasultDatas.find((item) => item.id === this.tabValue)
        return data && data.btns.hover.active
    }
    get graphData() {
        let data, cypherReasultData, graphData
        data = this.cypherStore.cypherReasultDatas.find((item) => item.id === this.tabValue)
        data && (cypherReasultData = data.cypherReasultData)
        cypherReasultData && (graphData = cypherReasultData.graphData)
        return graphData
    }
    get addEdgeSrcAndDst() {
        return this.graphData.addEdgeSrcAndDst
    }
    @Watch('graphData', { immediate: true, deep: true })
    onChangeGraphData(newVal: any, oldVal: any) {
        let nodes = this.graphData && this.graphData.graph.nodes
        let edges = this.graphData && this.graphData.graph.edges
        let cyNodes = this.createCyDatas(nodes || [], 'node')
        let cyEdges = this.createCyDatas(edges || [], 'edge')
        this.cyNodes = cyNodes
        this.cyEdges = cyEdges
        if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'upDateColor') {
            this.upDateColor(newVal.actionLog.options)
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'upDateSize') {
            this.upDateSize(newVal && newVal.actionLog.options)
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'upDateActiveProp') {
            this.upDateActiveProp(newVal.actionLog.options)
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'dbClickNode') {
            this.upDateByDbClick()
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'addNodes') {
            this.upDateByAddNode(newVal.actionLog)
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'addEdges') {
            this.upDateByAddEdge(newVal.actionLog)
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'editNodes') {
            this.upDateByEditNode(newVal.actionLog)
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'editEdges') {
            this.upDateByEditEdge(newVal.actionLog)
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'deleteNodes') {
            this.upDateByDeleteNode(newVal.actionLog)
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'deleteEdges') {
            this.upDateByDeleteEdge(newVal.actionLog)
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'refresh') {
            this.refresh()
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'upDateShow') {
            this.upDateShow(newVal.actionLog)
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'queryNeighborByCypher') {
            this.upDateByDbClick()
        } else if (newVal && newVal.actionLog && newVal.actionLog.actionName === 'upDateShowByFilter') {
            this.upDateShowByFilter(newVal.actionLog)
        }
    }
    created() {}
    mounted() {
        this.drawGraph()
        this.$cyEvents[this.tabValue] = {
            focusNode: (nodeId: string) => {
                this.focusNode(nodeId)
            },
            focusSourceOrEndNode: (status: boolean) => {
                this.focusSourceOrEndNode(status)
            },
            fixed: (status: boolean) => {
                status && this.fixed()
                !status && this.unfixed()
            },
            exportPNG: () => {
                this.exportPNG()
            },
            exportJSON: () => {
                this.exportJSON()
            },
            exportCSV: () => {
                this.exportCSV()
            },
            changeLayout: (data: string) => {
                this.cyLayout(data)
            }
        }
    }
    beforeDestroy() {
        if (this._layout) {
            this._layout.stop()
            this._layout = null
        }
        if (this._cy) {
            setTimeout(() => {
                this._cy.destroy()
                this._cy = null
            }, 1000)
        }
    }
    cyLayout(data: string) {
        this._layout.pon('layoutstop').then((event) => {
            if (data === 'grid') {
                this._layout = this._cy.layout(this.gridOption)
            } else if (data === 'force') {
                this._layout = this._cy.layout(this.forceOption)
            } else if (data === 'tree') {
                this._layout = this._cy.layout(this.treeOption)
            } else if (data === 'circle') {
                this._layout = this._cy.layout(this.circleOption)
            }
            this.currentLayoutOption = this[data + 'Option']
            this._layout.run()
            if (this.currentLayoutOption.name === 'euler') {
                this._cy.pan({
                    x: this._cyDom.clientWidth / 2,
                    y: this._cyDom.clientHeight / 2
                })
            }
        })
        this._layout.stop()
    }
    drawGraph() {
        console.time('draw-graph')

        let dom = this.$refs['resultGraphCanvas']
        let domWidth = dom && dom.clientWidth
        let domHeight = dom && dom.clientHeight
        let option = { ...CyConfig, ...{ container: dom } }
        option.elements.nodes = this.cyNodes
        option.elements.edges = this.cyEdges
        this._cy = cytoscape(option)
        this._eh = this._cy.edgehandles(ADDEDGEOPTION)
        this._layout = this._cy.layout(this.currentLayoutOption)
        this._cyDom = dom
        this.addEventToCy([
            {
                eventType: 'dblclick:timeout',
                eventTarget: 'cy',
                eventCallback: this.cyClick
            },
            {
                eventType: 'dblclick:timeout',
                eventTarget: 'node',
                eventCallback: this.nodeClick
            },
            {
                eventType: 'dblclick:timeout',
                eventTarget: 'edge',
                eventCallback: this.edgeClick
            },
            {
                eventType: 'dblclick',
                eventTarget: 'node',
                eventCallback: this.nodeDblClick
            },
            {
                eventType: 'mouseover',
                eventTarget: 'node',
                eventCallback: this.nodeMouseOver
            },
            {
                eventType: 'mouseout',
                eventTarget: 'node',
                eventCallback: this.nodeMouseOut
            },
            {
                eventType: 'mouseup',
                eventTarget: 'node',
                eventCallback: this.nodeMouseUp
            },
            {
                eventType: 'tapstart',
                eventTarget: 'node',
                eventCallback: this.nodeMouseDown
            },
            {
                eventType: 'cxttapend',
                eventTarget: 'node',
                eventCallback: this.cyCxttapend
            },
            {
                eventType: 'ehcomplete',
                eventTarget: 'cy',
                eventCallback: this.addEdgeComplete
            }
        ])
        this._layout.run()
        this._cy.fit()
        this._cy.zoom({
            level: this.zoomLevel
        })
        this._cy.pan({
            x: dom.clientWidth / 2,
            y: dom.clientHeight / 2
        })
        console.timeEnd('draw-graph')
    }
    focusNode(nodeId: string) {
        if (nodeId) {
            let node = this._cy.nodes(`#${nodeId}`)
            let label = node.data().sysPropties.label
            this.cypherStore.upDateActiveElement({
                tabValue: this.tabValue,
                data: [node.data()]
            })
            this._cy.center(node)
            node.addClass('active')
        }
    }
    focusSourceOrEndNode(status: boolean) {
        this.cypherStore.updateGraphDataAddEdgeSrcAndDst({ tabValue: this.tabValue, addEdgeSrcAndDst: [] })
        this.addEdgeStatus = status
        this._cy.remove(this._cy.edges('.add-new-edge'))
    }
    addEventToCy(
        events: Array<{
            eventType: string
            eventTarget: string
            eventCallback: Function
        }>
    ) {
        this._cy.dblclick(300)
        events.forEach((item: any) => {
            if (item.eventTarget === 'cy') {
                this._cy.on(item.eventType, item.eventCallback)
            } else {
                this._cy.on(item.eventType, item.eventTarget, item.eventCallback)
            }
        })
    }
    createCyDatas(data: Array<any>, type: string) {
        let newData: Array<any> = []
        let sdlList: Array<any> = []
        let mergeDataList: { [prop: string]: any } = {}
        data.forEach((item) => {
            let target = this.graphData && this.graphData.labels.find((l) => l.name === item.label && l.type === type)
            let size = target && target.size
            let color = target && target.color
            let borderColor = target && target.borderColor
            let displayName = target && target.activeProp
            let displayNameSource = target && target.activePropSource
            let cyData: cyData = {
                data: {
                    id: type === 'node' ? item.vid + '' : item.uid + '',
                    color: color,
                    size: size,
                    type: type === 'node' ? 'node' : 'edge',
                    properties: item.properties,
                    sysPropties: type === 'node' ? { vid: item.vid, label: item.label } : { uid: item.uid, label: item.label },
                    displayName: displayName,
                    displayNameSource: displayNameSource
                },
                classes: [item.label]
            }
            if (type === 'edge') {
                cyData.data.source = item.source + ''
                cyData.data.target = item.destination + ''
                let uid = item.uid.split('_')
                let sdl = uid[0] + '_' + uid[1] + '_' + uid[2]
                if (this.isMergeEdge) {
                    cyData.classes.push('merge')
                }
                !sdlList.includes(sdl) && sdlList.push(sdl) && cyData.classes.push('first')
                mergeDataList[sdl] = [...(mergeDataList[sdl] || []), ...[{ ...cyData.data }]]
            }
            if (type === 'node') {
                cyData.data.borderColor = borderColor
            }
            newData.push(cyData)
        })
        newData.forEach((item) => {
            if (item.data.type === 'edge') {
                let uid = item.data.id.split('_')
                let sdl = uid[0] + '_' + uid[1] + '_' + uid[2]
                item.data.mergeData = mergeDataList[sdl]
            }
        })
        return newData
    }
    upDateColor(data: any) {
        let selector = data.labelName
        let target
        if (data.type === 'node') {
            target = this._cy.nodes().filter((item) => item.data('sysPropties').label === selector)
        } else {
            target = this._cy.edges().filter((item) => item.data('sysPropties').label === selector)
        }
        target.forEach((item: any) => {
            item.data('color', data.color)
            item.data('mergeData') &&
                item.data('mergeData').forEach((item: any) => {
                    item.color = data.color
                })
            item.data('borderColor', data.borderColor)
        })
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateColor' })
    }
    upDateSize(data: any) {
        let selector = data.labelName
        let target
        if (data.type === 'node') {
            target = this._cy.nodes().filter((item) => item.data('sysPropties').label === selector)
        } else {
            target = this._cy.edges().filter((item) => item.data('sysPropties').label === selector)
        }
        target.forEach((item: any) => {
            item.data('size', data.size)
        })
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateSize' })
    }
    upDateActiveProp(data: any) {
        let selector = data.labelName
        let target
        if (data.type === 'node') {
            target = this._cy.nodes().filter((item) => item.data('sysPropties').label === selector)
        } else {
            target = this._cy.edges().filter((item) => item.data('sysPropties').label === selector)
        }
        target.forEach((item: any) => {
            item.data('displayName', data.propName)
            item.data('displayNameSource', data.propNameSource)
        })
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateActiveProp' })
    }
    upDateByDbClick(data?: any) {
        console.time('drawAddNode-dbclick')
        this._cy.nodes().forEach((item: any) => {
            item.lock()
        })
        this._cy.add({
            nodes: this.cyNodes,
            edges: this.cyEdges
        })
        this._layout.stop()
        this._cy
            .layout({ name: 'grid' })
            .run()
            .stop()
        this._layout = this._cy.layout(this.currentLayoutOption)
        this._layout.run()
        this._cy.fit()
        this._cy.zoom({
            level: this.zoomLevel
        })
        this._currenDbClickNode && this._cy.center(this._currenDbClickNode)
        this._cy.nodes().forEach((item: any) => {
            item.unlock()
            let id = item.data('id')
            let isFixedNode = this.fixNodeList[id]
            if (!isFixedNode) {
                item.scratch().euler && (item.scratch().euler.locked = false)
            }
        })
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateByDbClick' })
        console.timeEnd('drawAddNode-dbclick')
        if (this.fixedActive) {
            setTimeout(() => {
                this.fixed()
            }, 4000)
        }
    }
    upDateByAddNode(data: any) {
        console.time('drawAddNode-create')
        this._cy.nodes().forEach((item: any) => {
            item.lock()
        })
        this._cy.add({
            nodes: this.cyNodes
        })
        this._layout.stop()
        this._cy
            .layout({ name: 'grid' })
            .run()
            .stop()
        this._layout = this._cy.layout(this.currentLayoutOption)
        this._layout.run()
        this._cy.fit()
        this._cy.zoom({
            level: this.zoomLevel
        })
        let newNode = this._cy.$(`#${data.options.vid}`)
        newNode.addClass('hover')
        this._cy.center(newNode)
        this._cy.nodes().forEach((item: any) => {
            item.unlock()
            let id = item.data('id')
            let isFixedNode = this.fixNodeList[id]
            if (!isFixedNode) {
                item.scratch().euler && (item.scratch().euler.locked = false)
            }
        })
        this.panSource()
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateByAddNode' })
        console.timeEnd('drawAddNode-create')
    }
    upDateByAddEdge(data: any) {
        this._cy.add({
            edges: this.cyEdges
        })
        let mergeData = this._cy.edges('#' + data.options.uid).data().mergeData
        let uid = data.options.uid.split('_')
        let sdl = uid[0] + '_' + uid[1] + '_' + uid[2]
        this._cy.filter(`[ id ^= "${sdl}"]`).forEach((item: any) => {
            item.data('mergeData', mergeData)
        })
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateByAddEdge' })
    }
    upDateByEditNode(data: any) {
        let targetNode = this._cy.nodes('#' + data.options.vid)
        targetNode.data('properties', data.options.properties)
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateByEditNode' })
    }
    upDateByEditEdge(data: any) {
        let targetNode = this._cy.edges('#' + data.options.uid)
        targetNode.data('properties', data.options.properties)
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateByEditEdge' })
    }
    upDateByDeleteNode(data: any) {
        data.options.forEach((id: any) => {
            this._cy.remove('#' + id)
        })
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateByDeleteNode' })
    }
    upDateByDeleteEdge(data: any) {
        data.options.forEach((id: any) => {
            this._cy.remove('#' + id)
        })
        let uid = data.options[0].split('_')
        let sdl = uid[0] + '_' + uid[1] + '_' + uid[2]
        let mergeData = this.cyEdges[0].data.mergeData
        this._cy.filter(`[ id ^= "${sdl}"]`).forEach((item: any) => {
            item.data('mergeData', mergeData)
        })
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateByDeleteEdge' })
    }
    upDateShow(data: any) {
        let labelName = data.options.labelName
        let status = data.options.status
        let type = data.options.type
        if (!status) {
            type === 'node' && this._cy.nodes('.' + labelName).addClass('hidden')
            type === 'edge' && this._cy.edges('.' + labelName).addClass('hidden')
        } else {
            type === 'node' && this._cy.nodes('.' + labelName).removeClass('hidden')
            type === 'edge' && this._cy.edges('.' + labelName).removeClass('hidden')
        }
        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateShow' })
    }
    upDateShowByFilter(data: any) {
        let labelName = data.options.labelName
        let targetEle
        data.options.type === 'node' ? (targetEle = this._cy.nodes('.' + labelName)) : (targetEle = this._cy.edges('.' + labelName))
        targetEle.forEach((item) => {
            item.removeClass('hidden')
        })
        if (!data.options.clear) {
            if (data.options.sign === '=') {
                targetEle.forEach((item) => {
                    if (item.data('properties')[data.options.targetProp] !== data.options.targetValue) {
                        item.addClass('hidden')
                    }
                })
            } else if (data.options.sign === '>') {
                targetEle.forEach((item) => {
                    if (item.data('properties')[data.options.targetProp] <= data.options.targetValue) {
                        item.addClass('hidden')
                    }
                })
            } else if (data.options.sign === '<') {
                targetEle.forEach((item) => {
                    if (item.data('properties')[data.options.targetProp] >= data.options.targetValue) {
                        item.addClass('hidden')
                    }
                })
            } else if (data.options.sign === '>=') {
                targetEle.forEach((item) => {
                    if (item.data('properties')[data.options.targetProp] < data.options.targetValue) {
                        item.addClass('hidden')
                    }
                })
            } else if (data.options.sign === '<=') {
                targetEle.forEach((item) => {
                    if (item.data('properties')[data.options.targetProp] > data.options.targetValue) {
                        item.addClass('hidden')
                    }
                })
            }
        }

        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'upDateShowByFilter' })
    }
    mergeEdge() {
        if (this.isMergeEdge) {
            let sdlList: Array<any> = []
            this._cy.edges().forEach((ele: any) => {
                ele.addClass('merge')
                let uid = ele.data('id').split('_')
                let sdl = uid[0] + '_' + uid[1] + '_' + uid[2]
                !sdlList.includes(sdl) && sdlList.push(sdl) && ele.addClass('first')
            })
        } else {
            this._cy.edges().removeClass('merge', 'first')
        }
        this._cy.edges().removeClass('active')
        this.cypherStore.upDateActiveElement({
            tabValue: this.tabValue,
            data: []
        })
    }
    fixed() {
        this._layout && this._layout.stop()
    }
    unfixed() {
        this._cy.nodes().forEach((item: any) => {
            item.lock()
        })
        this._cy.layout({ name: 'grid' })
        this._layout = this._cy.layout(this.currentLayoutOption)
        this._layout.run()
        this._cy.nodes().forEach((item: any) => {
            item.unlock()
            let id = item.data('id')
            let isFixedNode = this.fixNodeList[id]
            if (!isFixedNode) {
                item.scratch().euler && (item.scratch().euler.locked = false)
            }
        })
    }
    panSource() {
        this._cy.pan({
            x: this._cyDom.clientWidth / 2,
            y: this._cyDom.clientHeight / 2
        })
    }
    exportPNG() {
        this.fixed()
        this._cy
            .png({
                full: true,
                output: 'blob-promise',
                maxWidth: 3656 * 5,
                maxHeight: 2644 * 5
            })
            .then((res: any) => {
                let name: Date = new Date()
                let fileName = 'graph图(' + name.toUTCString() + ')'
                downloadFile(fileName, res)
                this.unfixed()
            })
    }
    exportJSON() {
        let name: Date = new Date()
        let data = JSON.stringify(this.graphData.graph)
        // 如果查询结果没有节点，就将cypher的查询结果输出
        if (this.graphData.graph.nodes.length === 0) {
            let target: any = this.cypherStore.cypherReasultDatas.find((item) => item.id === this.tabValue)
            data = target.cypherReasultData.codeData.data.result
            data = JSON.stringify(data)
        }
        let fileName = 'graph图(' + name.toUTCString() + ').json'
        downloadFile(fileName, new Blob([data]))
    }
    exportCSV() {
        let name: Date = new Date()
        let fileName = 'graph图(' + name.toUTCString() + ').csv'
        let labels = this.graphData.labels
        let graph = this.graphData.graph
        let data = {
            nodes: {},
            edges: {}
        }
        if (graph.nodes.length > 0) {
            labels.forEach((item) => {
                if (item.name !== '*NODES' && item.name !== '*EDGES') {
                    let name = item.name
                    if (item.type === 'node') {
                        let content = ``
                        let header = ``
                        if (item.props.length) {
                            header = item.props.join(',') + ',vid' + '\n'
                        } else {
                            header = 'vid' + '\n'
                        }
                        let list = graph.nodes.filter((c) => c.label === name)
                        list.forEach((node) => {
                            item.props.forEach((p) => {
                                content += node.properties[p] + ','
                            })
                            content += node.vid + '\n'
                        })
                        data.nodes[name] = header + content
                    } else {
                        let content = ``
                        let header = ``
                        if (item.props.length) {
                            header = item.props.join(',') + ',uid,src,dst' + '\n'
                        } else {
                            header = 'uid,src,dst' + '\n'
                        }

                        let list = graph.edges.filter((c) => c.label === name)
                        list.forEach((edge) => {
                            item.props.forEach((p) => {
                                content += edge.properties[p] + ','
                            })
                            content += edge.uid + ',' + edge.source + ',' + edge.destination + '\n'
                        })
                        data.edges[name] = header + content
                    }
                }
            })
            let fileContentNodes = ``
            let fileContentEdges = ``
            Object.keys(data.nodes).forEach((k, index) => {
                if (index === 0) {
                    fileContentNodes += `LABEL=${k}\n${data.nodes[k]}`
                } else {
                    fileContentNodes += `${data.nodes[k]}`
                }
            })
            Object.keys(data.edges).forEach((k, index) => {
                if (index === 0) {
                    fileContentEdges += `LABEL=${k}\n${data.edges[k]}`
                } else {
                    fileContentEdges += `${data.edges[k]}`
                }
            })
            let file = 'NODES\n' + fileContentNodes + '\n' + 'EDGES\n' + fileContentEdges
            downloadFile(fileName, new Blob([file]))
        }
        // 如果查询结果没有节点，就将cypher的查询结果输出
        if (graph.nodes.length === 0) {
            let target: any = this.cypherStore.cypherReasultDatas.find((item) => item.id === this.tabValue)
            let data = target.cypherReasultData.codeData.data.result
            let content = ''
            data.forEach((row) => {
                content += row.toString() + '\n'
            })
            downloadFile(fileName, new Blob([content]))
        }
    }
    refresh() {
        this._cy.remove(this._cy.$())
        this._cy.resize()
        this._cy.add({
            nodes: this.cyNodes,
            edges: this.cyEdges
        })
        this._layout.stop()
        this._cy
            .layout({ name: 'grid' })
            .run()
            .stop()
        this._layout = this._cy.layout(this.currentLayoutOption)
        this._layout.run()
        this._cy.fit()
        this._cy.zoom({
            level: this.zoomLevel
        })

        if (this.currentLayoutOption.name === 'euler') {
            this._cy.pan({
                x: this._cyDom.clientWidth / 2,
                y: this._cyDom.clientHeight / 2
            })
        }

        this.cypherStore.upDateActionLogByEnd({ tabValue: this.tabValue, actionName: 'refresh' })
    }
    closeCxttapend() {
        let dom = this.$refs['workbenchCypherResultGraph']
        let child = dom.querySelector('#popperSearch')
        child && dom.removeChild(child)
    }
    //  ----- 鼠标交互事件Start -----
    // 单击[点]
    nodeClick(e: any) {
        let node = e.target
        if (this.addEdgeStatus) {
            this._eh.start(node)
        } else if (this.addEdgeSrcAndDst.length === 0) {
            let label = node.data().sysPropties.label
            this.cypherStore.upDateBtns({ tabValue: this.tabValue, active: false, index: 'add-node' })
            this.cypherStore.upDateBtns({ tabValue: this.tabValue, active: false, index: 'add-edge' })
            this.cypherStore.upDateActiveElement({
                tabValue: this.tabValue,
                data: [node.data()]
            })
            this._cy.$().removeClass('active')
            node.addClass('active')
        }
    }
    // 双击[点]
    nodeDblClick(e: any) {
        let id = e.target.data('id')
        this._currenDbClickNode = e.target
        this.cypherStore.queryByDbClick({
            graph: this.subGraphManageStore.selectedSubGraph,
            src: id,
            tabValue: this.tabValue
        })
    }
    // 鼠标悬停[点]
    nodeMouseOver(e: any) {
        let node = e.target
        node.addClass('hover')
        if (this.isHover) {
            node.connectedEdges().forEach((item: any) => {
                item.addClass('hover')
                item.connectedNodes((n: any) => {
                    n.addClass('hover')
                })
            })
        }
    }
    // 鼠标移出[点]
    nodeMouseOut(e: any) {
        this._cy.$('.hover').removeClass('hover')
    }
    // 鼠标抬起[点]
    nodeMouseUp(e: any) {
        let n = e.target.scratch()
        let id = e.target.data('id')
        n.euler && (n.euler.locked = true)
        this.fixNodeList[id] = e.target
    }
    // 鼠标右键[点]，出现菜单（写的太蠢，要重写）
    cyCxttapend(e: any) {
        let node = e.target
        let id = e.target.data('id')
        let n = e.target.scratch()
        n.euler && (n.euler.locked = true)
        this.fixNodeList[id] = e.target
        this._currenDbClickNode = e.target
        this.targetVertexID = node.data('sysPropties').vid
        let dom = this.$refs['workbenchCypherResultGraph']
        let nodePopper = document.getElementById('popperSearch')
        nodePopper && dom.removeChild(nodePopper)
        // 右键菜单代码后续需要优化 @诗源
        let popper = node.popper({
            content: () => {
                let div: HTMLDivElement = document.createElement('div')
                let content = this.$refs['cxttapend'].$el
                div.setAttribute('id', 'popperSearch')
                div.appendChild(content)
                dom.appendChild(div)
                return div
            },
            popper: {
                placement: 'right'
            }

            // renderedPosition: () => ({ x: position.x + 60, y: position.y - 60 })
        })
        let update = () => {
            popper.update()
        }
        node.on('position', update)
        this._cy.on('pan zoom resize', update)
    }
    // 鼠标按下[点]
    nodeMouseDown(e: any) {}
    // 点击边
    edgeClick(e: any) {
        if (this.addEdgeSrcAndDst.length === 0) {
            this.cypherStore.upDateBtns({ tabValue: this.tabValue, active: false, index: 'add-node' })
            this.cypherStore.upDateBtns({ tabValue: this.tabValue, active: false, index: 'add-edge' })
            let edge = e.target
            let data = e.target.data()
            let mergeData = e.target.data().mergeData
            this._cy.$().removeClass('active')
            edge.addClass('active')
            this.cypherStore.upDateActiveElement({
                tabValue: this.tabValue,
                data: !this.isMergeEdge ? [data] : mergeData
            })
        }
    }
    // 点击cy画布
    cyClick(e: any) {
        let target = e.target
        let isCy = !target.isNode && !target.isEdge
        e.stopPropagation()
        let dom = this.$refs['workbenchCypherResultGraph']
        let child = dom.querySelector('#popperSearch')
        if (isCy) {
            this.cypherStore.upDateBtns({ tabValue: this.tabValue, active: false, index: 'add-node' })
            if (!this.addEdgeStatus) {
                this.cypherStore.upDateBtns({ tabValue: this.tabValue, active: false, index: 'add-edge' })
                this.focusSourceOrEndNode(false)
            }
            this._cy
                .$()
                .removeClass('active')
                .removeClass('source')
                .removeClass('end')
            this.cypherStore.upDateActiveElement({
                tabValue: this.tabValue,
                data: []
            })
            setTimeout(() => {
                // 同时清空状态会使，mounted再次进入，原因未知，待确定，所以加一个延迟
                isCy &&
                    this.cypherStore.upDateActiveLabel({
                        tabValue: this.tabValue,
                        labelName: '',
                        labelType: ''
                    })
            }, 1)
        }
        child && dom.removeChild(child)
    }
    // 添加边完成
    addEdgeComplete(event, s, t, edge) {
        this.addEdgeStatus = false
        let srcNode = {
            id: s.data('id'),
            properties: s.data('properties'),
            sysPropties: s.data('sysPropties')
        }
        let dstNode = {
            id: t.data('id'),
            properties: t.data('properties'),
            sysPropties: t.data('sysPropties')
        }
        let edgeData = {
            id: edge.data('id')
        }
        let data = [srcNode, dstNode, edgeData]
        this.cypherStore.updateGraphDataAddEdgeSrcAndDst({ tabValue: this.tabValue, addEdgeSrcAndDst: data })
    }

    //  ----- 鼠标交互事件End -----
}
</script>
<style lang="less" scoped>
.workbench-cypher-result-graph {
    position: absolute;
    width: 100%;
    height: 100%;
    background: #ffffff;
    z-index: 2;
    .cxttpaend .lalala {
        // 临时展示用
        position: absolute;
        left: 100px;
        top: 100px;
        z-index: 999;
    }
    .result-graph-canvas {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        overflow: hidden;
    }
}
</style>
