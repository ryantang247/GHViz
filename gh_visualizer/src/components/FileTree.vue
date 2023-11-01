<template>
  <div>
    <h1>GitHub File Tree Visualization</h1>
    <div v-if="fileTree!=null">
      <button @click="renderTreemap">Render Treemap</button>
    </div>
  </div>
</template>

<script>

//remember add a watch so that everytime we query the data is able to change
import * as d3 from 'd3';
export default {
  name: 'FileTree',
  props: {
    fileTree: {}
  },
  methods: {
    addChildren(node) {
      // Check if the node is an object and not null
      if (typeof node === 'object' && node !== null) {
        // Initialize children property as an empty array
        node.children = [];

        // Recursively add children to each child node
        for (const key in node) {
          if (Object.prototype.hasOwnProperty.call(node, key)) {
            // Skip the 'children' property itself to avoid infinite recursion
            if (key !== 'children') {
              // Add a 'name' property to each child node based on the key
              const child = node[key];
              child.name = key;

              // Recursively add children to the child node
              this.addChildren(child);

              // Add the child to the 'children' array
              node.children.push(child);
            }
          }
        }
      }
    },
    renderTreemap() {

      //convert
      const json = JSON.stringify(this.fileTree)
      this.addChildren(json)
      console.log("converted json", json)

      const margin = { top: 40, right: 10, bottom: 10, left: 10 };
      const width = 960 - margin.left - margin.right;
      const height = 500 - margin.top - margin.bottom;
      const color = d3.scaleOrdinal().range(d3.schemeCategory20c);

      const treemap = d3.treemap().size([width, height]);

      const div = d3.select(this.$refs.treemapContainer)
          .style('position', 'relative')
          .style('width', `${width + margin.left + margin.right}px`)
          .style('height', `${height + margin.top + margin.bottom}px`)
          .style('left', `${margin.left}px`)
          .style('top', `${margin.top}px`);

      // d3.json('flare.json', (error, data) => {
      //   if (error) throw error;

        const root = d3.hierarchy(json, (d) => d.children).sum((d) => d.size);
        const tree = treemap(root);

        const node = div.datum(root).selectAll('.node')
            .data(tree.leaves())
            .enter().append('div')
            .attr('class', 'node')
            .style('left', (d) => `${d.x0}px`)
            .style('top', (d) => `${d.y0}px`)
            .style('width', (d) => `${Math.max(0, d.x1 - d.x0 - 1)}px`)
            .style('height', (d) => `${Math.max(0, d.y1 - d.y0 - 1)}px`)
            .style('background', (d) => color(d.parent.data.name))
            .text((d) => d.data.name);

        d3.selectAll('input').on('change', function change() {
          const value = this.value === 'count' ? (d) => (d.size ? 1 : 0) : (d) => d.size;
          const newRoot = d3.hierarchy(json, (d) => d.children).sum(value);

          node.data(treemap(newRoot).leaves())
              .transition()
              .duration(1500)
              .style('left', (d) => `${d.x0}px`)
              .style('top', (d) => `${d.y0}px`)
              .style('width', (d) => `${Math.max(0, d.x1 - d.x0 - 1)}px`)
              .style('height', (d) => `${Math.max(0, d.y1 - d.y0 - 1)}px`);
        });
      // });
    },
  }
};
</script>

<style scoped>
body {
  font-family: "Helvetica Neue", Helvetica, Arial, sans-serif;
  margin: auto;
  position: relative;
  width: 960px;
}
form {
  position: absolute;
  right: 10px;
  top: 10px;
}
.node {
  border: solid 1px white;
  font: 10px sans-serif;
  line-height: 12px;
  overflow: hidden;
  position: absolute;
  text-indent: 2px;
}
</style>
