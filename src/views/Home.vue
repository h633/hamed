<template>
  <div class="container">
    <div id="tree-graph">

    </div>
  </div>
</template>

<script>
  import * as d3 from 'd3'
  export default {
    name: 'Home',
    data() {
      return {
        tree: {
          name: "father",
          children:[{
            name: "son1",
            children:[ {name: "grandson"}, {name: "grandson2"}]
          },{
            name: "son2",
            children:[ {name: "grandson3"}, {name: "grandson4"}]
          }]
        },
        width: null,
        height : null,
        margin : {
          top: 0,
          right: 90,
          bottom: 30,
          left: 90,
        }
      }
    },
    mounted() {
      this.Tree()
    },
    methods: {
      Tree(){
        // Set the dimensions and margins of the diagram
        this.width = 960 - this.margin.left - this.margin.right;
        this.height = 500 - this.margin.top - this.margin.bottom;

        // append the svg object to the body of the page
        // appends a 'group' element to 'svg'
        // moves the 'group' element to the top left margin
        let svg = d3
                .select("#tree-graph")
                .append("svg")
                .attr("width", this.width + this.margin.right + this.margin.left)
                .attr("height", this.height + this.margin.top + this.margin.bottom)
                .append("g")
                .attr("transform", "translate(" + this.margin.left + "," + this.margin.top + ")");

        let i = 0,
                duration = 750,
                root;

        // declares a tree layout and assigns the size
        let treemap = d3.tree().size([this.height, this.width]);

        // Assigns parent, children, height, depth
        root = d3.hierarchy(this.tree, (d) => {
          return d.children;
        });
        root.x0 = this.height / 2;
        root.y0 = 0;

        update(root);

        // Collapse the node and all it's children
        function  update(source) {
          // Assigns the x and y position for the nodes
          let treeData = treemap(root);

          // Compute the new tree layout.
          let nodes = treeData.descendants(),
                  links = treeData.descendants().slice(1);

          // Normalize for fixed-depth.
          nodes.forEach((d) => {
            d.y = d.depth * 180;
          });

          // ****************** Nodes section ***************************

          // Update the nodes...
          let node = svg.selectAll("g.node").data(nodes, (d) => {
            return d.id || (d.id = ++i);
          });

          // Enter any new modes at the parent's previous position.
          let nodeEnter = node
                  .enter()
                  .append("g")
                  .attr("class", "node")
                  .attr("transform", (e, d) => {
                    return "translate(" + source.y0 + "," + source.x0 + ")";
                  })
                  .on("click", function (e, d) {
                    if (d.children) {
                      d._children = d.children;
                      d.children = null;
                    } else {
                      d.children = d._children;
                      d._children = null;
                    }
                    update(d);
                  });

          // Add Circle for the nodes
          nodeEnter
                  .append("circle")
                  .attr("class", "node")
                  .attr("r", 1e-6)
                  .style("fill", (d) => {
                    return d._children ? "lightsteelblue" : "#fff";
                  });

          // Add labels for the nodes
          nodeEnter
                  .append("text")
                  .attr("dy", ".35em")
                  .attr("x", function(d) {
                    return d.children || d._children ? -13 : 13;
                  })
                  .attr("text-anchor", function(d) {
                    return d.children || d._children ? "end" : "start";
                  })
                  .text((d) => {
                    return d.data.name;
                  });

          // UPDATE
          let nodeUpdate = nodeEnter.merge(node);

          // Transition to the proper position for the node
          nodeUpdate
                  .transition()
                  .duration(duration)
                  .attr("transform",(d) => {
                    return "translate(" + d.y + "," + d.x + ")";
                  });

          // Update the node attributes and style
          nodeUpdate
                  .select("circle.node")
                  .attr("r", 10)
                  .style("fill", (d) => {
                    return d._children ? "lightsteelblue" : "#fff";
                  })
                  .attr("cursor", "pointer");

          // Remove any exiting nodes
          let nodeExit = node
                  .exit()
                  .transition()
                  .duration(duration)
                  .attr("transform", (d) => {
                    return "translate(" + source.x + "," + source.y + ")";
                  })
                  .remove();

          // On exit reduce the node circles size to 0
          nodeExit.select("circle").attr("r", 1e-6);

          // On exit reduce the opacity of text labels
          // nodeExit.select("text").style("fill-opacity", 1e-6);

          // ****************** links section ***************************

          // Update the links...
          let link = svg.selectAll("path.link").data(links, (d) => {
            return d.id;
          });

          // Enter any new links at the parent's previous position.
          let linkEnter = link
                  .enter()
                  .insert("path", "g")
                  .attr("class", "link")
                  .attr("d", (d) => {
                    let o = {
                      x: source.x0,
                      y: source.y0,
                    };
                    return diagonal(o, o);
                  });

          link
                  .enter()
                  .insert("text", "g")
                  .attr("font-family", "Arial, Helvetica, sans-serif")
                  .attr("fill", "Orange")
                  .style("font", "normal 12px Arial")
                  .attr("transform", (d) => {
                    return (
                            "translate(" +
                            (d.parent.x + d.x) / 2 +
                            "," +
                            (d.parent.y + d.y) / 2 +
                            ")"
                    );
                  })
                  .attr("dy", ".35em")
                  .attr("text-anchor", "middle")
                  .text((d) => {
                    return d.data.name;
                  });

          // UPDATE
          let linkUpdate = linkEnter.merge(link);

          // Transition back to the parent element position
          linkUpdate
                  .transition()
                  .duration(duration)
                  .attr("d", (d) => {
                    return diagonal(d, d.parent);
                  });

          // Remove any exiting links
          let linkExit = link
                  .exit()
                  .transition()
                  .duration(duration)
                  .attr("d", (d) => {
                    let o = {
                      x: source.x,
                      y: source.y,
                    };
                    return diagonal(o, o);
                  })
                  .remove();

          // Store the old positions for transition.
          nodes.forEach((d) => {
            d.x0 = d.x;
            d.y0 = d.y;
          });

          // Creates a curved (diagonal) path from parent to the child nodes
          function diagonal(s, d) {
            let path = `M ${s.y} ${s.x}
              C ${(s.y + d.y) / 2} ${s.x},
                ${(s.y + d.y) / 2} ${d.x},
                ${d.y} ${d.x}`;

            return path;
          }
        }
      }
    }
  }
</script>
<style>

</style>