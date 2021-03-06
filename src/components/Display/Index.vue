<!-- This displays all the items in the database as a list view -->

<template>
  <v-container fluid
    grid-list-md>
    <search-filters></search-filters>
    <v-layout>
      <v-flex d-flex
        xs12
        sm6
        md6
        offset-sm3>
        <!-- Search bar to search submission by item type -->
        <search-panel></search-panel>
      </v-flex>
    </v-layout>
    <!--Sets the design and displays Lost Items-->
    <list :collectionCluster="clusteredCollections" />
    <h1 v-if="!clusteredCollections">
      Loading...
    </h1>
    <v-layout mt-4>
      <v-flex xs12>
        <div v-if="nbPages > 1"
          class="text-xs-center">
          <v-pagination v-model="page"
            :length="nbPages"
            prev-icon="icon-left-open"
            next-icon="icon-right-open"></v-pagination>
        </div>
      </v-flex>
    </v-layout>
    <div id="powered-by-algolia">
      <p>Powered by</p>
      <a href='https://www.algolia.com/'
        target="_blank"
        rel="noopener noreferrer">
        <img src="../../../static/svg/algolia.svg"
          width="24"
          height="24"
          alt="">
      </a>
    </div>
  </v-container>
</template>

<script>
import List from './List'
import SearchPanel from './SearchPanel'
import SearchFilters from './SearchFilters/Index'
import { mapGetters, mapActions } from 'vuex'

// Creates a reference to firebase storag

export default {
  components: {
    List,
    SearchPanel,
    SearchFilters
  },
  data () {
    return {
      clusteredCollections: null,
      page: 1,
      filtersObject: {}
    }
  },
  computed: {
    ...mapGetters([
      'queriedItems',
      'nbPages',
      'queryDate',
      'queryCategory',
      'queryTime',
      'queryStatus'
    ]),
    breakpoint () { return this.$vuetify.breakpoint },
    pageZeroIndexed () {
      return this.page - 1
    },
    filters () {
      let computedFilters = ''
      const keys = Object.keys(this.filtersObject)
      for (let keyIndex = 0; keyIndex < keys.length; keyIndex++) {
        let propertyName = keys[keyIndex]
        if (computedFilters.length > 0) {
          computedFilters += ' AND '
        }
        if (propertyName === 'time') {
          const startTime = parseInt(this.filtersObject.time.timeStart)
          const endTime = parseInt(this.filtersObject.time.timeEnd)
          if (startTime > endTime) {
            computedFilters += `time: ${endTime} TO ${startTime}`
          } else {
            computedFilters += `time: ${startTime} TO ${endTime}`
          }
        } else if (propertyName === 'collection') {
          let status = '('
          if (this.filtersObject.collection.lost) {
            status = 'collection: lost'
          }
          if (this.filtersObject.collection.found) {
            if (status.length > 1) {
              status += 'OR collection: found)'
            } else {
              status += 'collection: found)'
            }
          }
          computedFilters += status
        } else {
          computedFilters += `${propertyName}:${this.filtersObject[propertyName]}`
        }
      }
      return computedFilters
    }
  },
  methods: {
    ...mapActions([
      'updateCollectionQuery'
    ]),
    updateCluster (collection) {
      if (!collection || !collection.length) {
        this.clusteredCollections = []
      }
      let group = []
      let limit = 0
      let cluster = 0
      let clusters = []
      if (!this.breakpoint) {
        limit = 1
      } else {
        if (this.breakpoint.width < 500) {
          limit = 1
        } else if (this.breakpoint.width < 612) {
          limit = 2
        } else if (this.breakpoint.width < 885) {
          limit = 3
        } else {
          limit = 4
        }
      }
      collection.forEach((submission, index) => {
        group.push(submission)
        cluster++
        if (cluster === limit || collection.length - 1 === index) {
          clusters.push(group)
          group = []
          cluster = 0
        }
      })
      this.clusteredCollections = clusters
    }
  },
  watch: {
    breakpoint () {
      this.updateCluster([...this.queriedItems])
    },
    '$route.query.search': {
      immediate: true,
      async handler (query) {
        this.updateCollectionQuery({ query, page: this.pageZeroIndexed, filters: this.filters })
      }
    },
    queriedItems (collection) {
      this.updateCluster([...this.queriedItems])
    },
    page () {
      const query = this.$route.query.search
      this.updateCollectionQuery({ query, page: this.pageZeroIndexed, filters: this.filters })
    },
    queryDate (queryDate) {
      this.$set(this.filtersObject, 'date', queryDate)
    },
    queryCategory (queryCategory) {
      if (!queryCategory && this.filtersObject.hasOwnProperty('category')) {
        this.$delete(this.filtersObject, 'category')
        return
      }
      this.$set(this.filtersObject, 'category', `"${queryCategory}"`)
    },
    queryTime (queryTime) {
      this.$set(this.filtersObject, 'time', queryTime)
    },
    queryStatus (queryStatus) {
      if (queryStatus.found && queryStatus.lost) {
        this.$delete(this.filtersObject, 'collection')
        return
      }
      this.$set(this.filtersObject, 'collection', queryStatus)
    },
    filters (filters) {
      const query = this.$route.query.search
      this.updateCollectionQuery({ query, page: this.pageZeroIndexed, filters })
    }
  }
}
</script>
<style scoped>
p {
  display: inline-block;
  font-weight: 600;
}
img {
  margin-top: 10px;
}
#powered-by-algolia {
  text-align: center;
}
</style>
