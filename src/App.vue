<template>
<div >
  <vs-row vs-w="12">
    <vs-col vs-type="flex" vs-justify="center" vs-align="center" vs-lg="8" vs-sm="8" vs-xs="12" >
      <div id="tree">
      </div>
    </vs-col>
    <vs-col vs-type="flex" vs-justify="center" vs-align="flex-start" vs-lg="4" vs-sm="4" vs-xs="12">
       <vs-card actionable class="cardx" v-if="activeNode" >
        <div slot="header" class="text-center">
          <h3>
            {{ activeNode.name }}
          </h3>
        </div>
        <div slot="media" class="node-image">
          <img :src="activeNode.image">
        </div>
      </vs-card>
    </vs-col>
  </vs-row>
</div>
</template>

<script>
import * as d3 from 'd3'
import axios from 'axios'

export default {
    name: 'App',
    data () {
        return {
            loadData : null,
            width : window.innerWidth,
            height : window.innerHeight,
            boxWidth : 250,
            boxHeight : 20,
            gap : {
                width: 100,
                height: 12
            },
            margin : {
                top: 16,
                right: 16,
                bottom: 16,
                left: 16
            },
            Nodes: [],
            links: [],
            activeNode : null,
            loading: true,
            errored: false,
            activeImage: null,
        }
    },
    async created (){
        await this.getData()
        await this.draw_chart() 
    },
    methods: {
        getData: async function() {
            return axios.get('http://127.0.0.1:8000/data/')
                .then((response) => {
                    console.log("data", response.data)
                    this.loadData = response.data
                }).catch((error) => {
                    console.log(error)
                    this.errored = true
                });
        },
        find(text) {
            var i;
            for (i = 0; i < this.Nodes.length; i += 1) {
                if (this.Nodes[i].name === text) {
                    return this.Nodes[i];
                }
            }
            return null;
        },
        unvisite_links() {
            "use strict";
            this.links.forEach(function (d) {
                d.visited = false;
            });
        },
        mouse_action(val, stat, direction) {
            "use strict";
            d3.select("#" + val.id).classed("active", stat);
            
            this.links.forEach((d) => {
                if (direction == "root") {
                    if (d.source.id === val.id) {
                        d3.select("#" + d.id).classed("activelink", stat); // change link color
                        d3.select("#" + d.id).classed("link", !stat); // change link color
                        if (d.target.lvl < val.lvl)
                            this.mouse_action(d.target, stat, "left");
                        else if (d.target.lvl > val.lvl)
                            this.mouse_action(d.target, stat, "right");
                    }
                    if (d.target.id === val.id) {
                        d3.select("#" + d.id).classed("activelink", stat); // change link color
                        d3.select("#" + d.id).classed("link", !stat); // change link color
                        if (direction == "root") {
                            if(d.source.lvl < val.lvl)
                                this.mouse_action(d.source, stat, "left");
                            else if (d.source.lvl > val.lvl)
                                this.mouse_action(d.source, stat, "right");
                        }
                    }
                }else if (direction == "left") {
                    if (d.source.id === val.id && d.target.lvl < val.lvl) {
                        d3.select("#" + d.id).classed("activelink", stat); // change link color
                        d3.select("#" + d.id).classed("link", !stat); // change link color

                        this.mouse_action(d.target, stat, direction);
                    }
                    if (d.target.id === val.id && d.source.lvl < val.lvl) {
                        d3.select("#" + d.id).classed("activelink", stat); // change link color
                        d3.select("#" + d.id).classed("link", !stat); // change link color
                        this.mouse_action(d.source, stat, direction);
                    }
                }else if (direction == "right") {
                    if (d.source.id === val.id && d.target.lvl > val.lvl) {
                        d3.select("#" + d.id).classed("activelink", stat); // change link color
                        d3.select("#" + d.id).classed("link", !stat); // change link color
                        this.mouse_action(d.target, stat, direction);
                    }
                    if (d.target.id === val.id && d.source.lvl > val.lvl) {
                        d3.select("#" + d.id).classed("activelink", stat); // change link color
                        d3.select("#" + d.id).classed("link", !stat); // change link color
                        this.mouse_action(d.source, stat, direction);
                    }
                }
            });
        },
        renderRelationshipGraph(data) {
            "use strict";
            var count = []
            var svg;
            svg = d3.select("#tree").append("svg")
                .attr("width", 800)
                .attr("height", 800)
                .attr("viewBox","0 0 1000 1000")
                .append("g");

            var line = d3.line()
                .x(function(d) {
                    return d.y;
                })
                .y(function(d) {
                    return d.x;
                })

            data.Nodes.forEach(function (d) {
                count[d.lvl] = 0;
            });

            data.Nodes.forEach((d, i) => {
                d.x = this.margin.left + d.lvl * (this.boxWidth + this.gap.width);
                d.y = this.margin.top + (this.boxHeight + this.gap.height) * count[d.lvl];
                d.id = "n" + i;
                count[d.lvl] += 1;
                this.Nodes.push(d);
            });

            data.links.forEach((d) => {
                this.links.push({
                    source: this.find(d.source),
                    target: this.find(d.target),
                    id: "l" + this.find(d.source).id + this.find(d.target).id
                });
            });

            this.unvisite_links();

            svg.append("g")
                .attr("class", "nodes");

            var node = svg.select(".nodes")
                .selectAll("g")
                .data(this.Nodes)
                .enter()
                .append("g")
                .attr("class", "unit");

            node.append("rect")
                .attr("x", function (d) { return d.x; })
                .attr("y", function (d) { return d.y; })
                .attr("id", function (d) { return d.id; })
                .attr("width", this.boxWidth)
                .attr("height", this.boxHeight)
                .attr("class", "node")
                .attr("rx", 6)
                .attr("ry", 6)
                .on("click", (d) => {
                    if (this.activeNode) {
                        this.mouse_action(this.activeNode, false, "root");
                    }
                    this.mouse_action(d3.select(d.target).datum(), false, "root");
                    this.mouse_action(d3.select(d.target).datum(), true, "root");
                    console.log("212", this.activeNode, d3.select(d.target).datum())
                    this.activeNode = d3.select(d.target).datum()
                    this.unvisite_links();
                })
            node.append("text")
                .attr("class", "label")
                .attr("x", function (d) { return d.x + 14; })
                .attr("y", function (d) { return d.y + 15; })
                .text(function (d) { return d.name; });
            
            this.links.forEach((li) => {
                svg.append("path", "g")
                    .attr("class", "link")
                    .attr("id", li.id)
                    .attr("d", () => {
                        var oTarget = {
                            x: li.target.y + 0.5 * this.boxHeight,
                            y: li.target.x
                        };
                        var oSource = {
                            x: li.source.y + 0.5 * this.boxHeight,
                            y: li.source.x
                        };
                        
                        if (oSource.y < oTarget.y) {
                            oSource.y += this.boxWidth;
                        } else {
                            oTarget.y += this.boxWidth;
                        }
                        return line([oSource, oTarget])
                    });
            });
        },
        draw_chart() {
            var data = {"Nodes": [], "links": []}
            this.loadData.songs.forEach(item => {
                data['Nodes'].push({"lvl":0, "name": item.name, "image": item.image})
                item.wiki_items.forEach(witem => {
                    data['links'].push({"source": item.name, "target": witem.name},)
                    witem.recommendations.forEach(ritem => {
                        data['links'].push({"source": witem.name, "target": ritem.name},)
                })
                })
                item.facebook_friends.forEach(fitem => {
                    data['links'].push({"source": item.name, "target": fitem.name},)
                    fitem.recommendations.forEach(ritem => {
                        data['links'].push({"source": fitem.name, "target": ritem.name},)
                })
                })
                item.twitter_experts.forEach(titem => {
                    data['links'].push({"source": item.name, "target": titem.name},)
                    titem.recommendations.forEach(ritem => {
                        data['links'].push({"source": titem.name, "target": ritem.name},)
                })
                })
            });
            this.loadData.witems.forEach(item => {
                data['Nodes'].push({"lvl":1, "name": item.name, "image": item.image})
            });
            this.loadData.fitems.forEach(item => {
                data['Nodes'].push({"lvl":1, "name": item.name, "image": item.image})
            });
            this.loadData.titems.forEach(item => {
                data['Nodes'].push({"lvl":1, "name": item.name, "image": item.image})
            });
            this.loadData.recos.forEach(item => {
                data['Nodes'].push({"lvl":2, "name": item.name, "image": item.image})
            });

            console.log("data", data)
            this.renderRelationshipGraph(data);
        }
    }
}
</script>

<style>
rect {
    fill: #CCC;
    cursor: pointer;
}
.active {
    fill: orange;
    stroke: orange;
}

.activelink {
    fill: none;
    stroke: orange;
    stroke-width: 2.5px;
}

.label {
    fill: white;
    font-family: sans-serif;
    pointer-events: none;
}
.link {
    fill: none;
    /* stroke: #000; */
    stroke-width: 2.5px;
}
.node-image {
    height: 500px;
}
</style>
