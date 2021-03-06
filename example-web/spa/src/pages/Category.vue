<template>
  <v-layout row wrap class="px-2">
    <v-flex xs12>
      <vue-crud-x ref="category" :parentId="parentId" v-bind="categoryDefs" />
    </v-flex>
  </v-layout>
</template>

<script>
// import { http } from '@/axios'
import { apolloClient } from '@/graphql'
import { GET_CATEGORIES, GET_CATEGORY, PATCH_CATEGORY, CATEGORY_UPDATED, POST_CATEGORY } from '@/queries'
import { PAGESIZE, PAGESIZE_OPTS } from '@/config'

export default {
  name: 'category',
  mounted() {
    this.subscriptionObserver = apolloClient.subscribe({ query: CATEGORY_UPDATED }).subscribe(
      data => {
        console.log('Data', data)
      },
      e => console.error('Error', e.toString())
    )  
  },
  beforeDestroy() {
    if (this.subscriptionObserver) this.subscriptionObserver.unsubscribe()
  },
  data () {
    return {
      subscriptionObserver: null,
      parentId: null,
      categoryDefs: {
        pageSize: PAGESIZE,
        pageSizeOptions: PAGESIZE_OPTS,
        title: 'Category',
        vtable: {
          headers: [
            { text: 'Category id', value: 'id', class: 'pa-1' },
            { text: 'Category Name', value: 'name', class: 'pa-1', render: (value) => value }
          ]
        },
        filters: null,
        form: {
          'id': {
            type: 'v-text-field',
            value: '',
            default: '',
            hidden: 'add', // add, edit, all, null
            readonly: 'all', // add, edit, all, null
            validation: null, // validation function no in place yet
            'field-wrapper': { xs12: true, sm6: true },
            'field-input': {
              label: 'ID'
            }
          },
          'name': {
            type: 'v-text-field',
            value: '',
            default: '',
            'field-wrapper': { xs12: true, sm6: true },
            'field-input': {
              label: 'Name',
              rules: [v => !!v || 'Item is required']
            }
          }
        },
        crud: {
          find: async ({ pagination = {}, filters = {}, sorters = {} }) => {
            let records = []
            let totalRecords = 0
            const { page, itemsPerPage } = pagination
            // GrqphQL
            try {
              const rv = await apolloClient.query({ query: GET_CATEGORIES, variables: { page: page > 0 ? page - 1 : 0 ,  limit: parseInt(itemsPerPage) } })
              // console.log('ABC', rv.data.getCategories)
              records = rv.data.getCategories.results
              totalRecords = rv.data.getCategories.total
              return { status: 200, data: { records, totalRecords } }
            } catch (e) {
              console.log(e)
              return { status: 500, error: e.toString() }
            }
            // REST
            // try {
            //   const { data: { results, total } } = await http.get('/api/categories', {
            //     params: {
            //       page: page > 0 ? page - 1 : 0,
            //       limit: itemsPerPage
            //     }
            //   })
            //   // console.log(results)
            //   records = results
            //   totalRecords = total
            //   return { status: 200, data: { records, pagination, totalRecords } }
            // } catch (e) {
            //   return { status: e.response.status, error: e.toString() }
            // }
          },
          findOne: async (id) => {
            // GrqphQL
            try {
              const rv = await apolloClient.query({ query: GET_CATEGORY, variables: { id: parseInt(id) } })
              return { status: 200, data: rv.data.getCategory }
            } catch (e) {
              return { status: 500, error: e.toString() }
            }
            // REST
            // try {
            //   const { data } = await http.get(`/api/categories/${id}`)
            //   return { status: 200, data }
            // } catch (e) {
            //   return { status: e.response.status, error: e.toString() }
            // }
          },
          create: async (payload) => {
            try {
              // TBD handle errors in refetch
              let { record: { id, ...noIdData } } = payload
              await apolloClient.mutate({
                mutation: POST_CATEGORY,
                variables: {
                  body: {
                    name: noIdData.name
                  }
                },
                // use in memory cache
                update: (cache, { data: { postCategory } }) => {
                  const data = cache.readQuery({ query: GET_CATEGORIES, variables: { page: 0, limit: 2 } })
                  // work with the cache data - START
                  // if you are working with paging, you may want to comment the lines below out if page limit already reached. e.g
                  // if (this.$refs.category.totalRecords < data.getCategories.total) {
                  alert('skip this if page limit. otherwise you get extra line!')
                  data.getCategories.results.push(postCategory)
                  data.getCategories.total += 1
                  // }
                  // work with the cache data - END
                  cache.writeQuery({ query: GET_CATEGORIES, data })
                },
                // data is updated immediately - why so troublesome? - WRITE TO UI FIRST BEFORE GETTING SERVER RESPONSE
                // CAVEAT - Not all fields may be updated on client... e.g. server timestamps
                // if mutation fails too bad
                optimisticResponse: {
                  __typename: 'Mutation',
                  postCategory: {
                    __typename: 'Category',
                    id: -1, // set as invalid number so no clashes with existing
                    body: {
                      name: noIdData.name
                    }
                  }
                },
                // refetchQueries - rerun queries after mutation and update the store
                // Not applicable for now as vue-crud-x reloads data after create/update/delete
                // Can set poll export default graphql(channelsListQuery, { options: { pollInterval: 5000 }, })(ChannelsList);
                refetchQueries: [
                  { query: GET_CATEGORIES, variables: { page: 0, limit: 2 } }
                  // add additional queries here, e.g.
                  // { query: GET_SOMETHING_RELATED, variables: { pageNum: 1, pageSize: 2 } }
                ]
              })
              return { status: 201, data: null }
            } catch (e) {
              return { status: 500, error: e.toString() }
            }
            // try {
            //   let { record: { id, ...noIdData } } = payload
            //   // const rv =
            //   const { data } = await http.post('/api/categories', noIdData)
            //   return { status: 201, data }
            // } catch (e) {
            //   return { status: e.response.status, error: e.toString() }
            // }
          },
          update: async (payload) => {
            // GrqphQL
            try {
              let { record: { id, ...noIdData } } = payload
              console.log(noIdData)
              // const rv =
              await apolloClient.mutate({
                mutation: PATCH_CATEGORY,
                variables: {
                  id: parseInt(id),
                  body: {
                    name: noIdData.name
                  }
                }
              })
              // console.log('rv', rv)
              return { status: 200, data: payload.record }
            } catch (e) {
              return { status: 500, error: e.toString() }
            }
            // REST
            // try {
            //   let { record: { id, ...noIdData } } = payload
            //   const { data } = await http.patch(`/api/categories/${id}`, noIdData)
            //   return { status: 200, data }
            // } catch (e) {
            //   return { status: e.response.status, error: e.toString() }
            // }
          } // done
        }
      }
    }
  }
}
</script>
