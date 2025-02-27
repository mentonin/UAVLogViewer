<template>
    <div id="line" ref="line" style="width:100%;height: 100%"></div>
</template>

<script>
import Plotly from 'plotly.js'
import {store} from './Globals.js'
import * as d3 from 'd3'

let Color = require('color')

let timeformat = ':02,2f'
let annotationsEvents = []
let annotationsModes = []
let annotationsParams = []

const updatemenus = [
    {
        active: 0,
        buttons: [
            {
                args: ['annotations', annotationsModes],
                label: 'Nothing',
                method: 'relayout'
            },
            {
                args: ['annotations', [...annotationsEvents, ...annotationsModes]],
                label: 'Events',
                method: 'relayout'
            },
            {
                args: ['annotations', [...annotationsEvents, ...annotationsModes, ...annotationsParams]],
                label: 'Events + Params',
                method: 'relayout'
            }
        ],
        direction: 'left',
        pad: {'r': 10, 't': 10},
        showactive: true,
        type: 'buttons',
        x: 0.1,
        xanchor: 'left',
        y: 1.2,
        yanchor: 'top'
    }
]

let plotOptions = {
    legend: {
        x: 0.1,
        y: 1,
        traceorder: 'normal',
        borderwidth: 1
    },
    showlegend: true,
    // eslint-disable-next-line
    plot_bgcolor: '#f8f8f8',
    // eslint-disable-next-line
    paper_bgcolor: 'white',
    // autosize: true,
    margin: {t: 20, l: 0, b: 30, r: 10},
    xaxis: {
        title: 'Time since boot',
        domain: [0.15, 0.85],
        rangeslider: {},
        tickformat: timeformat
    },
    yaxis: {
        // title: 'axis1',
        titlefont: {
            color: '#1f77b4'
        },
        tickfont: {
            color: '#1f77b4', size: 12
        },
        anchor: 'free',
        position: 0.03,
        autotick: true,
        showline: true,
        ticklen: 3,
        tickangle: 45
    },
    yaxis2: {
        // title: 'yaxis2 title',
        titlefont: {color: '#ff7f0e'},
        tickfont: {color: '#ff7f0e', size: 12},
        anchor: 'free',
        overlaying: 'y',
        side: 'left',
        position: 0.07,
        autotick: true,
        showline: true,
        ticklen: 3,
        tickangle: 45
    },
    yaxis3: {
        // title: 'yaxis4 title',
        titlefont: {color: '#2ca02c'},
        tickfont: {color: '#2ca02c'},
        anchor: 'free',
        overlaying: 'y',
        side: 'left',
        position: 0.11,
        autotick: true,
        showline: true,
        ticklen: 3,
        tickangle: 45
    },
    yaxis4: {
        // title: 'yaxis5 title',
        titlefont: {color: '#d62728'},
        tickfont: {color: '#d62728'},
        anchor: 'free',
        overlaying: 'y',
        side: 'left',
        position: 0.92,
        autotick: true,
        showline: true,
        ticklen: 3,
        tickangle: 45
    },
    yaxis5: {
        // title: 'yaxis5 title',
        titlefont: {color: '#9467BD'},
        tickfont: {color: '#9467BD'},
        anchor: 'free',
        overlaying: 'y',
        side: 'left',
        position: 0.96,
        autotick: true,
        showline: true,
        ticklen: 3,
        tickangle: 45
    },
    yaxis6: {
        // title: 'yaxis5 title',
        titlefont: {color: '#8C564B'},
        tickfont: {color: '#8C564B'},
        anchor: 'free',
        overlaying: 'y',
        side: 'left',
        position: 1.0,
        autotick: true,
        showline: true,
        ticklen: 3,
        tickangle: 45
    }

}

export default {
    created () {
        this.$eventHub.$on('cesium-time-changed', this.setCursorTime)
        this.$eventHub.$on('force-resize-plotly', this.resize)
        this.zoomInterval = null
        this.cache = {}
    },
    mounted () {
        const WIDTH_IN_PERCENT_OF_PARENT = 90
        d3.select('#line')
            .append('div')
            .style({
                width: '100%',
                'margin-left': (100 - WIDTH_IN_PERCENT_OF_PARENT) / 2 + '%',
                height: '100%'
            })

        this.gd = d3.select('#line').node()
        let _this = this
        this.$nextTick(function () {
            if (this.$route.query.hasOwnProperty('ranges')) {
                let ranges = []
                for (let field of this.$route.query.ranges.split(',')) {
                    ranges.push(parseFloat(field))
                }
                if (ranges.length > 0) {
                    plotOptions.xaxis.range = [ranges[0], ranges[1]]
                }
                if (ranges.length >= 4) {
                    plotOptions.yaxis.range = [ranges[2], ranges[3]]
                }
                if (ranges.length >= 6) {
                    plotOptions.yaxis2.range = [ranges[4], ranges[5]]
                }
                if (ranges.length >= 8) {
                    plotOptions.yaxis3.range = [ranges[6], ranges[7]]
                }
                if (ranges.length >= 10) {
                    plotOptions.yaxis4.range = [ranges[8], ranges[9]]
                }
            }
            if (this.$route.query.hasOwnProperty('plots')) {
                for (let field of this.$route.query.plots.split(',')) {
                    _this.addPlots([field])
                }
            }
        })
        this.instruction = ''
        this.$eventHub.$on('togglePlot', this.togglePlot)
        this.$eventHub.$on('addPlots', this.addPlots)
        this.$eventHub.$on('plot', this.plot)
        this.$eventHub.$on('clearPlot', this.clearPlot)
    },
    beforeDestroy () {
        this.$eventHub.$off('animation-changed')
        this.$eventHub.$off('cesium-time-changed')
        this.$eventHub.$off('addPlots')
        this.$eventHub.$off('hidePlot')
        this.$eventHub.$off('togglePlot')
        clearInterval(this.interval)
    },
    data () {
        return {
            gd: null,
            plotInstance: null,
            state: store
        }
    },
    methods: {
        csvButton () {
            return [
                {
                    name: 'downloadCsv',
                    title: 'Download data as csv',
                    icon: Plotly.Icons.disk,
                    click: () => {
                        console.log(this.gd.data)
                        var data = this.gd.data
                        let header = ['timestamp(ms)']
                        for (let series of data) {
                            header.push(series.name.split(' |')[0])
                        }

                        let indexes = []

                        let interval = 100
                        let lasttime = Infinity
                        let finaltime = 0

                        for (let series in data) {
                            indexes.push(0)
                            const x = data[series].x
                            lasttime = Math.min(lasttime, x[0])
                            finaltime = Math.max(finaltime, x[x.length - 1])
                        }
                        let csv = [header]
                        while (lasttime < finaltime - interval) {
                            const line = [lasttime]
                            for (let series in data) {
                                let index = indexes[series]
                                let x = data[series].x[index]
                                while (x < lasttime) {
                                    indexes[series] += 1
                                    index = indexes[series]
                                    x = data[series].x[index]
                                }
                                line.push(data[series].y[index])
                            }
                            csv.push(line)
                            lasttime = lasttime + interval
                        }
                        let csvContent = csv.map(e => e.join(',')).join('\n')
                        var blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' })
                        var link = document.createElement('a')
                        var url = URL.createObjectURL(blob)
                        link.setAttribute('href', url)
                        link.setAttribute('download', 'data.csv')
                        link.style.visibility = 'hidden'
                        document.body.appendChild(link)
                        link.click()
                        document.body.removeChild(link)
                    }
                }
            ]
        },
        resize () {
            Plotly.Plots.resize(this.gd)
        },
        waitForMessages (messages) {
            for (let message of messages) {
                this.$eventHub.$emit('loadType', message)
            }
            let interval
            let _this = this
            let counter = 0
            return new Promise((resolve, reject) => {
                interval = setInterval(function () {
                    for (let message of messages) {
                        if (!_this.state.messages.hasOwnProperty(message)) {
                            counter += 1
                            if (counter > 30) { // 30 * 300ms = 9 s timeout
                                console.log('not resolving')
                                clearInterval(interval)
                                reject(new Error(`Could not load messageType ${message}`))
                            }
                            return
                        }
                    }
                    clearInterval(interval)
                    resolve()
                }, 300)
            })
        },
        onRangeChanged (event) {
            this.addMaxMinMeanToTitles()
            if (event !== undefined) {
                // this.$router.push({query: query})
                if (event.hasOwnProperty('xaxis.range')) {
                    this.state.timeRange = event['xaxis.range']
                }
                if (event.hasOwnProperty('xaxis.range[0]')) {
                    this.state.timeRange = [event['xaxis.range[0]'], event['xaxis.range[1]']]
                }
                if (event.hasOwnProperty('xaxis.autorange')) {
                    this.state.timeRange = [this.gd.layout.xaxis.range[0], this.gd.layout.xaxis.range[1]]
                }
            }
        },
        addMaxMinMeanToTitles   () {
            let average = arr => arr.reduce((p, c) => p + c, 0) / arr.length
            var gd = this.gd
            var xRange = gd.layout.xaxis.range

            var needsRelayout = false

            gd.data.forEach(trace => {
                var len = Math.min(trace.x.length, trace.y.length)
                var xInside = []
                var yInside = []

                for (var i = 0; i < len; i++) {
                    var x = trace.x[i]
                    var y = trace.y[i]

                    if (x > xRange[0] && x < xRange[1]) {
                        xInside.push(x)
                        yInside.push(y)
                    }
                }
                const extraData = ` | Min: ${Math.min(...yInside).toFixed(2)} \
    Max: ${Math.max(...yInside).toFixed(2)} \
    Mean: ${average(yInside).toFixed(2)}`

                if (trace.name.indexOf(extraData) === -1) {
                    trace.name = trace.name.split(' | ')[0] + extraData
                    needsRelayout = true
                }
            })
            if (needsRelayout) {
                Plotly.relayout(this.gd, this.gd.layout)
            }
        },
        isPlotted (fieldname) {
            for (let field of this.state.expressions) {
                if (field.name === fieldname) {
                    return true
                }
            }
            return false
        },
        getFirstFreeAxis () {
            // get free axis number
            for (let i of this.state.allAxis) {
                let taken = false
                for (let field of this.state.expressions) {
                    // eslint-disable-next-line
                    if (field.axis == i) {
                        taken = true
                    }
                }
                if (!taken) {
                    return i
                }
            }
            return this.state.allAxis.length - 1
        },
        getFirstFreeColor () {
            // get free color
            for (let i of this.state.allColors) {
                let taken = false
                for (let field of this.state.expressions) {
                    // eslint-disable-next-line
                    if (field.color == i) {
                        taken = true
                    }
                }
                if (!taken) {
                    return i
                }
            }
            return this.state.allColors[this.state.allColors.length - 1]
        },
        createNewField (fieldname, axis, color) {
            if (color === undefined) {
                color = this.getFirstFreeColor()
            } else if (!isNaN(color)) {
                color = this.state.allColors[color]
            }
            if (axis === undefined) {
                axis = this.getFirstFreeAxis()
            }
            return {
                name: fieldname,
                color: color,
                axis: axis
            }
        },

        addPlots (plots) {
            this.state.plotLoading = true
            let requested = new Set()
            const RE = /[A-Z][A-Z0-9_]+(\[[0-9]\])?\.[a-zA-Z0-9]+/g
            let RE2 = /[A-Z][A-Z0-9_]+(\[[0-9]\])/g
            for (let plot of plots) {
                let expression = plot[0]
                // ensure we have the data
                let messages = expression.match(RE)
                // not match ATT, GPS
                messages = expression.match(RE2)
                if (messages !== null) {
                    for (const message of messages) {
                        if (!(message in this.state.messages)) {
                            if (requested.has(message)) {
                                continue
                            }
                            console.log('missing message type: ' + message)
                            requested.add(message)
                        }
                    }
                }
            }
            if ([...requested].length > 0) {
                console.log([...requested])
                this.waitForMessages([...requested]).then(() => {
                    this.addPlots(plots)
                })
                    .catch((e) => {
                        alert(e)
                        this.plot()
                    })
                return
            }
            let newplots = []
            for (let plot of plots) {
                let expression = plot[0]
                let axis = plot[1]
                let color = plot[2]
                if (!this.isPlotted(expression)) {
                    newplots.push(this.createNewField(expression, axis, color))
                }
            }
            this.state.expressions.push(...newplots)
        },
        removePlot (fieldname) {
            var index = this.state.expressions.indexOf(fieldname) // <-- Not supported in <IE9
            if (index !== -1) {
                this.state.expressions = this.state.expressions.splice(index, 1)
            }
            this.plot()
            if (this.state.expressions.length === 0) {
                this.state.plotOn = false
            }
            this.onRangeChanged()
        },
        clearPlot () {
            while (this.state.expressions.length) {
                this.state.expressions.pop()
            }
            this.state.expressions.lenght = 0
        },
        resetAxis (index) {
            // Resets the Y axis so that the next plot autoranges
            // unfortunately the axis are named yaxis, yaxis2, yaxis3... and so on
            let suffix = ''
            suffix = index === 0 ? suffix : parseInt(index) + 1
            let key = 'yaxis' + suffix
            let obj = {}
            // Use older dict and set autorange to true
            obj[key] = plotOptions[key]
            obj[key].autorange = true
            Plotly.relayout(this.gd, obj)
        },
        togglePlot (fieldname, axis, color, silent) {
            if (this.isPlotted((fieldname))) {
                let index
                for (let i in this.state.expressions) {
                    if (this.state.expressions[i].name === fieldname) {
                        index = i
                    }
                }
                this.resetAxis(this.state.expressions[index].axis)
                this.state.expressions.splice(index, 1)
                if (this.state.expressions.length === 0) {
                    this.state.plotOn = false
                }
                this.onRangeChanged()
            } else {
                this.addPlots([[fieldname, axis, color]])
            }
            console.log(this.state.expressions)
            // if (silent !== true) {
            //     this.plot()
            //     this.state.plotLoading = false
            // }
        },
        calculateXAxisDomain () {
            let start = 0.02
            let end = 0.98
            for (let field of this.state.expressions) {
                if (field.axis === 0) {
                    start = Math.max(start, 0.03)
                } else if (field.axis === 1) {
                    start = Math.max(start, 0.07)
                } else if (field.axis === 2) {
                    start = Math.max(start, 0.11)
                } else if (field.axis === 5) {
                    end = Math.min(end, 0.96)
                } else if (field.axis === 4) {
                    end = Math.min(end, 0.92)
                } else if (field.axis === 3) {
                    end = Math.min(end, 0.88)
                }
            }
            return [start, end]
        },
        getAxisTitle (fieldAxis) {
            let names = []
            for (let field of this.state.expressions) {
                if (field.axis === fieldAxis) {
                    names.push(field.name)
                }
            }
            return names.join(', ')
        },
        findMessagesInExpression (expression) {
            let RE = /(?<message>[A-Z][A-Z0-9_]+(\[[0-9]\])?)(\.(?<field>[A-Za-z0-9_]+))?/g
            let match = []
            for (let m of expression.matchAll(RE)) {
                match.push([m.groups.message, m.groups.field])
            }
            return match
        },
        expressionCanBePlotted (expression, reask = false) {
            // TODO: USE this regex with lookahead once firefox supports it
            // let RE = /(?<!\.)\b[A-Z][A-Z0-9_]+\b/g
            // let fields = expression.name.match(RE)
            let messages = this.findMessagesInExpression(expression.name)

            if (messages === null) {
                return [true, '']
            }
            for (let [message, field] of messages) {
                if ((!(message in this.state.messageTypes) && !((message + '[0]') in this.state.messageTypes))) {
                    console.log('ERROR: attempted to plot unavailable message: ' + message)
                    this.state.plotLoading = false
                    if (reask) {
                        this.$eventHub.$emit('loadType', message)
                    }
                    return [false, `invalid message: ${message}`]
                }
                if (field !== undefined) {
                    if (field !== 'time_boot_ms' && this.state.messageTypes[message].expressions.indexOf(field) < 0) {
                        console.log('ERROR: attempted to plot unavailable field: ' + field)
                        return [false, `invalid field: ${message}.${field}`]
                    }
                }
                console.log(message + ' is plottable')
            }
            return [true, '']
        },
        messagesAreAvailable (messages) {
            // TODO: USE this regex with lookahead once firefox supports it
            // let RE = /(?<!\.)\b[A-Z][A-Z0-9_]+\b/g
            // let fields = expression.name.match(RE)
            for (let message of messages) {
                if (!(message in this.state.messages) || this.state.messages[message].lenght === 0) {
                    if (!((message + '[0]') in this.state.messages) ||
                        this.state.messages[message + '[0]'].lenght === 0) {
                        return false
                    }
                }
            }
            return true
        },
        evaluateExpression (expression1) {
            let start = new Date()
            if (expression1 in this.cache) {
                console.log('HIT: ' + expression1)
                return this.cache[expression1]
            }
            console.log('MISS! evaluating : ' + expression1)
            // TODO: USE this regex with lookahead once firefox supports it
            // let RE = /(?<!\.)\b[A-Z][A-Z0-9_]+\b/g
            let fields = this.findMessagesInExpression(expression1).map(field => field[0])
            fields = fields === null ? [] : fields
            let messages = fields.length !== 0 ? (fields) : []
            // use time of first message for now
            let x
            if (messages.length > 0) {
                x = this.state.messages[messages[0]].time_boot_ms
            } else {
                try {
                    x = this.state.messages['ATT'].time_boot_ms
                } catch {
                    x = this.state.messages['ATTITUDE'].time_boot_ms
                }
            }
            // used to find the corresponding time indexes between messages
            let timeIndexes = new Array(fields.length).fill(0)
            let y = []
            let expression = expression1
            // eslint-disable-next-line
            for (let field in fields) {
                if (isNaN(field)) {
                    break
                }
                expression = expression.replaceAll(fields[field], 'a[' + field + ']')
            }
            let f
            try {
                // eslint-disable-next-line
                f = new Function('a', 'return ' + expression)
            } catch (e) {
                return {'error': e}
            }
            for (let time of x) {
                let vals = []
                for (let fieldIndex in timeIndexes) { // array of indexes, one for each field
                    while (this.state.messages[messages[fieldIndex]].time_boot_ms[timeIndexes[fieldIndex]] < time) {
                        timeIndexes[fieldIndex] += 1
                    }
                    const newobj = {}
                    for (let key of Object.keys(this.state.messages[messages[fieldIndex]])) {
                        newobj[key] = this.state.messages[messages[fieldIndex]][key][timeIndexes[fieldIndex]]
                    }
                    vals.push(newobj)
                }
                try {
                    y.push(f(vals))
                } catch (e) {
                    return {'error': e}
                }
            }
            console.log('evaluated ' + expression)
            let data = this.addGaps({
                x: x,
                y: y
            })
            this.cache[expression1] = data
            console.log('Evaluation took ' + (new Date() - start) + 'ms')
            return data
        },
        addGaps (data) {
            // Creates artifical gaps in order to break lines in plot when messages are not being received
            let newData = {x: [], y: [], isSwissCheese: false}
            let lastx = data.x[0]
            let totalPoints = data.x.length
            let totalGaps = 0
            for (let i = 0; i < data.x.length; i++) {
                if ((data.x[i] - lastx) > 3000) {
                    newData.x.push(data.x[i] - 1)
                    newData.y.push(null)
                    totalGaps += 1
                }
                newData.x.push(data.x[i])
                newData.y.push(data.y[i])
                lastx = data.x[i]
            }
            if (totalGaps > (totalPoints / 2)) {
                newData.isSwissCheese = true
            }
            return newData
        },
        plot () {
            console.log('plot()')
            plotOptions.title = this.state.file
            let _this = this
            let datasets = []
            this.state.expressionErrors = []
            let errors = []

            for (let expression of this.state.expressions) {
                let [canplot, error] = this.expressionCanBePlotted(expression, false)
                if (!canplot) {
                    errors.push(error)
                    this.state.expressionErrors = errors
                    return
                }
                errors.push(null)
            }

            let messages = []
            for (let expression of this.state.expressions) {
                messages = [...messages, ...(this.findMessagesInExpression(expression.name).map(message => message[0]))]
            }
            if (!this.messagesAreAvailable(messages)) {
                this.waitForMessages(messages).then(this.plot)
                    .catch((e) => {
                        alert(e)
                        this.plot()
                    })
            }

            for (let expression of this.state.expressions) {
                let data = this.evaluateExpression(expression.name)
                if ('error' in data) {
                    this.state.expressionErrors.push(data['error'])
                    data = {x: 0, y: 0}
                } else {
                    this.state.expressionErrors.push(null)
                }
                console.log(data)
                let mode = data.isSwissCheese ? 'lines+markers' : 'lines'

                let regularMarker = {
                    size: 4,
                    color: expression.color
                }

                let crossMarker = {
                    size: 5,
                    symbol: 'cross-thin',
                    color: expression.color,
                    line: {
                        color: expression.color,
                        width: 1
                    }
                }
                let marker = data.isSwissCheese ? crossMarker : regularMarker
                datasets.push({
                    name: expression.name,
                    // type: 'scattergl',
                    mode: mode,
                    x: data.x,
                    y: data.y,
                    yaxis: 'y' + (expression.axis + 1),
                    line: {
                        color: expression.color,
                        width: 1.5
                    },
                    marker: marker
                })
                let axisname = expression.axis > 0 ? ('yaxis' + (expression.axis + 1)) : 'yaxis'

                if (expression.axis <= 6) {
                    plotOptions[axisname].title = {
                        text: this.getAxisTitle(expression.axis),
                        font: {
                            color: expression.color
                        }
                    }
                    plotOptions[axisname].tickfont.color = expression.color
                    /* if (this.state.messageTypes[msgtype].complexFields[msgfield].units !== '?') {
                         plotOptions[axisname].title.text +=
                            ' (' + this.state.messageTypes[msgtype].complexFields[msgfield].units + ')'
                    } */
                }
            }
            let start = new Date()
            console.log('starting plotting itself...')

            let plotData = datasets

            plotOptions.xaxis = {
                rangeslider: {},
                domain: this.calculateXAxisDomain(),
                title: 'time_boot (ms)',
                tickformat: ':04,2f'
            }
            if (this.plotInstance !== null) {
                plotOptions.xaxis.range = this.gd._fullLayout.xaxis.range
                Plotly.newPlot(this.gd, plotData, plotOptions, {scrollZoom: true, responsive: true})
            } else {
                this.plotInstance = Plotly.newPlot(
                    this.gd,
                    plotData,
                    plotOptions,
                    {
                        modeBarButtonsToAdd: this.csvButton(),
                        scrollZoom: true,
                        editable: true,
                        responsive: true
                    }
                )
            }
            console.log('plotting done in ' + (new Date() - start) + 'ms')
            start = new Date()
            this.gd.on('plotly_relayout', this.onRangeChanged)
            this.gd.on('plotly_hover', function (data) {
                let infotext = data.points.map(function (d) {
                    return d.x
                })
                _this.$eventHub.$emit('hoveredTime', infotext[0])
            })

            this.addModeShapes()
            this.addEvents()
            this.addParamChanges()

            this.state.plotLoading = false

            let bglayer = document.getElementsByClassName('bglayer')[0]
            let rect = bglayer.childNodes[0]
            this.cursor = document.createElementNS('http://www.w3.org/2000/svg', 'line')
            let x = rect.getAttribute('x')
            let y = rect.getAttribute('y')
            let y2 = parseInt(y) + parseInt(rect.getAttribute('height'))
            this.cursor.setAttribute('id', 'batata')
            this.cursor.setAttribute('x1', x)
            this.cursor.setAttribute('y1', y)
            this.cursor.setAttribute('x2', x)
            this.cursor.setAttribute('y2', y2)
            this.cursor.setAttribute('stroke-width', 1)
            this.cursor.setAttribute('stroke', 'black')
            bglayer.append(this.cursor)
            console.log('layout done in ' + (new Date() - start) + 'ms')
        },
        setCursorTime (time) {
            try {
                let bglayer = document.getElementsByClassName('bglayer')[0]
                let rect = bglayer.childNodes[0]
                let x = parseInt(rect.getAttribute('x'))
                let width = parseInt(rect.getAttribute('width'))
                let percTime = (time - this.gd.layout.xaxis.range[0]) /
                    (this.gd.layout.xaxis.range[1] - this.gd.layout.xaxis.range[0])
                let newx = x + width * percTime
                this.cursor.setAttribute('x1', newx)
                this.cursor.setAttribute('x2', newx)
            } catch (err) {
                console.log(err)
            }
        },
        getMode (time) {
            for (let mode in this.state.flightModeChanges) {
                if (this.state.flightModeChanges[mode][0] > time) {
                    if (mode - 1 < 0) {
                        return this.state.flightModeChanges[0][1]
                    }
                    return this.state.flightModeChanges[mode - 1][1]
                }
            }
            return this.state.flightModeChanges[this.state.flightModeChanges.length - 1][1]
        },
        getModeColor (time) {
            return this.state.cssColors[this.setOfModes.indexOf(this.getMode(time))]
        },
        darker (color) {
            return Color(color).darken(0.2).string()
        },
        addModeShapes () {
            let shapes = []
            let modeChanges = [...this.state.flightModeChanges]
            modeChanges.push([this.gd.layout.xaxis.range[1], null])

            for (let i = 0; i < modeChanges.length - 1; i++) {
                shapes.push(
                    {
                        type: 'rect',
                        // x-reference is assigned to the x-values
                        xref: 'x',
                        // y-reference is assigned to the plot paper [0,1]
                        yref: 'paper',
                        x0: modeChanges[i][0],
                        y0: 0,
                        x1: modeChanges[i + 1][0],
                        y1: 1,
                        fillcolor: this.getModeColor(modeChanges[i][0] + 1),
                        opacity: 0.15,
                        line: {
                            width: 0
                        }
                    }
                )
            }
            Plotly.relayout(this.gd, {
                shapes: shapes
            })
        },
        addEvents () {
            annotationsEvents = []
            let i = -300
            for (let event of this.state.events) {
                annotationsEvents.push(
                    {
                        xref: 'x',
                        yref: 'paper',
                        x: event[0],
                        y: 0,
                        yanchor: 'bottom',
                        text: event[1].toLowerCase(),
                        showarrow: true,
                        arrowwidth: 1,
                        arrowcolor: '#999999',
                        ay: i,
                        ax: 0
                    }
                )
                i += 23
                if (i > 0) {
                    i = -300
                }
            }
            let modeChanges = [...this.state.flightModeChanges]
            modeChanges.push([this.gd.layout.xaxis.range[1], null])
            for (let i = 0; i < modeChanges.length - 1; i++) {
                annotationsModes.push(
                    {
                        xref: 'x',
                        // y-reference is assigned to the plot paper [0,1]
                        yref: 'paper',
                        x: modeChanges[i][0],
                        xanchor: 'left',
                        y: 0,
                        textangle: 90,
                        text: '<b>' + modeChanges[i][1] + '</b>',
                        showarrow: false,
                        font: {
                            color: this.darker(this.getModeColor(modeChanges[i][0] + 1))
                        },
                        opacity: 1
                    }
                )
            }
            Plotly.relayout(this.gd, {
                annotations: annotationsModes,
                updatemenus: updatemenus
            })
            updatemenus[0].buttons[0].args = ['annotations', annotationsModes]
            updatemenus[0].buttons[1].args = ['annotations', [...annotationsEvents, ...annotationsModes]]
            updatemenus[0].buttons[2].args = ['annotations', [...annotationsEvents, ...annotationsModes,
                ...annotationsParams]]
        },
        addParamChanges () {
            let i = -300
            annotationsParams = []
            let firstFetch = new Set()
            let startAt = null
            for (let change of this.state.params.changeArray) {
                if (!firstFetch.has(change[1])) {
                    firstFetch.add(change[1])
                } else {
                    startAt = change[0]
                    break
                }
            }
            let last = [0, 0]
            for (let change of this.state.params.changeArray) {
                if (change[0] < startAt) {
                    continue
                }
                // This takes care of repeated param changed logs we get for some reason
                if (change[2] === last[2] && change[1] === last[1]) {
                    continue
                }
                // Filter some "noisy" parameters
                if (['STAT_FLTTIME', 'STAT_RUNTIME'].includes(change[1])) {
                    continue
                }
                last = change
                annotationsParams.push(
                    {
                        xref: 'x',
                        yref: 'paper',
                        x: change[0],
                        y: 0,
                        yanchor: 'bottom',
                        text: change[1] + '->' + change[2].toFixed(4),
                        showarrow: true,
                        arrowwidth: 1,
                        arrowcolor: '#999999',
                        ay: i,
                        ax: 0
                    }
                )
                i += 23
                if (i > 0) {
                    i = -300
                }
            }
            updatemenus[0].active = 0
            Plotly.relayout(this.gd, {
                annotations:
                [
                    ...annotationsModes
                ],
                updatemenus: updatemenus
            })
            updatemenus[0].buttons[2].args =
            [
                'annotations',
                [
                    ...annotationsEvents,
                    ...annotationsModes,
                    ...annotationsParams
                ]
            ]
        }
    },
    computed: {
        setOfModes () {
            let set = []
            for (let mode of this.state.flightModeChanges) {
                if (!set.includes(mode[1])) {
                    set.push(mode[1])
                }
            }
            return set
        },
        timeRange () {
            if (this.state.timeRange != null) {
                return this.state.timeRange
            }
            return undefined
        },
        expressions () {
            return this.state.expressions
        }
    },
    watch: {
        timeRange (range) {
            if (Math.abs(this.gd.layout.xaxis.range[0] - range[0]) > 5 ||
                    Math.abs(this.gd.layout.xaxis.range[1] - range[1]) > 5) {
                if (this.zoomInterval !== null) {
                    clearTimeout(this.zoomInterval)
                }
                this.zoomInterval = setTimeout(() => {
                    Plotly.relayout(this.gd, {
                        xaxis: {
                            title: 'Time since boot',
                            range: range,
                            domain: this.calculateXAxisDomain(),
                            rangeslider: {},
                            tickformat: timeformat
                        }
                    })
                }, 500)
            }
            return range // make linter happy, it says this is a computed property(?)
        },
        expressions: {
            deep: true,
            handler () {
                this.plot()
            }
        }
    }
}

</script>
<style>
    .js-plotly-plot {
        margin-left: 0 !important;
    }

    .shapelayer path {
        pointer-events: none !important;
    }
</style>
