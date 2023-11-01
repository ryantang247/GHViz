<!-- RepoInfo.vue -->

<template>
  <div>
    <label>
      Username:
      <input v-model="username" />
    </label>
    <label>
      Repository:
      <input v-model="repo" />
    </label>
    <button @click="getFileContents">Fetch Repository Info</button>

    <div v-if="loading">Loading...</div>
    <div v-else>
    </div>
    <div v-if="errorMessage!=null">Error fetching data</div>
    <div v-if="loaded">
    <FileTree v-bind:file-tree="filePaths" />
    </div>
  </div>
  <GhAPI/>
</template>

<script>
import gql from 'graphql-tag';
import FileTree from "@/components/FileTree.vue";
import {Octokit} from "@octokit/rest";
import GhAPI from "@/components/GitAPI.vue";

export default {
  components: {GhAPI, FileTree},
  // components: {FileTree},
  data() {
    return {
      // by default, all data types ret
      loaded : false,
      loading: false,
      username: 'ShiqiYu',
      repo: 'CPP',
      filePaths: {},
      currentCommit : {},
      repository: {},
      errorMessage: null,
      overall_tree: {}
    };
  },
  methods: {
    handleCommitClick(commit) {
      // Handle the click event for a commit
      this.currentCommit = commit.oid
      console.log('Clicked commit:', this.currentCommit);

    },
    async getFileContents() {
      this.loading = true;
      const octokit = new Octokit({
        auth: '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'
      })

      // const apiUrl = `https://api.github.com/repos/${this.username}/${this.repo}/contents/`;

      try {
        const response = await octokit.request('GET /repos/{owner}/{repo}/git/trees/{branch}?recursive=3', {
          owner: this.username,
          repo: this.repo,
          branch: 'main',
          headers: {
            'Accept': 'application/vnd.github.v3.raw', // This header indicates that you want the raw content
          },
        });

        // remember to put loaded to true for passing data

        const content = response.data;
        console.log(content)
        this.loading = false;
        this.loaded  =true;
        this.filePaths = this.buildDirectoryTree(content.tree);
        return content;

        } catch{
          this.loading = false;
          this.error = 'Error fetching data!'
        }

      },
      buildDirectoryTree(files) {
        const tree = {};

        files.forEach(file => {
          const pathSegments = file.path.split('/');
          let currentNode = tree;

          pathSegments.forEach(segment => {
            currentNode[segment] = currentNode[segment] || {};
            currentNode = currentNode[segment];
          });
        });

        return tree;
      },

    updateTree(existingTree, filePath) {
      const pathSegments = filePath.split('/');
      let currentNode = existingTree;

      // Traverse the existing tree based on the path segments
      pathSegments.forEach(segment => {
        currentNode[segment] = currentNode[segment] || {};
        currentNode = currentNode[segment];
      });

      // You can set additional properties or values to the final node if needed
      currentNode.isModified = true;

      return existingTree;
    },

    convertJSON (filePath){
        const filePaths = filePath.map(item => item.path);
        const output = {};

        for (const path of filePaths) {
          let current = output;
          const segments = path.split('/').filter(segment => segment !== ''); // Filter out empty segments

          for (const segment of segments) {
            if (!(segment in current)) {
              current[segment] = {};
            }

            current = current[segment];
          }
        }

        return output;
    },

    async fetchRepositoryInfo() {
      this.loading = true;

      try {
        // Execute the GraphQL query with dynamic variables
        const result = await this.$apollo.query({
          query: gql`
            query ($owner: String!, $name: String!) {
              repository(owner: $owner, name: $name) {
                pullRequests(first: 100, states: [OPEN ,CLOSED, MERGED]) {
                  nodes {
                    title
                    number
                    state
                    changedFiles
                    files(first: 100) {
                      nodes {
                        path
                        additions
                        deletions

                      }
                    }
                  }
                }
              }
            }
          `,
          variables: {
            owner: "ryantang247",
            name: "Dl-project",

            // by default, the branch we use is main, edit this later
            expression: "main"
          },
        });
        // console.log(result.data.repository)
        this.loaded = true
        this.repository = result.data.repository;

        // this.overall_tree = this.buildDirectoryTree()

        // gets each commit
        // this.fileStruct = result.data.repository.commits.target.history.edges
        // console.log("File structure: ")
        console.log((this.repository))
      } catch (error) {
        console.error('Error fetching repository info:', error);
      } finally {
        this.loading = false;
      }
    }
    },
};
</script>

<style scoped>
.clickable-commit {
  cursor: pointer;
  color: blue;
  text-decoration: underline;
  margin-bottom: 5px;
}
/* Add more styles as needed */
</style>
