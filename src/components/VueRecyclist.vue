<template>
  <div class="vue-recyclist">
    <div ref="list" class="vue-recyclist-items" :style="{height: height + 'px'}">
      <div v-for="(item, index) in visibleItems" class="vue-recyclist-item" :style="{transform: 'translate3d(0,' + item.top + 'px,0)'}">
        <div v-show="tombstone" :class="{'vue-recyclist-transition': tombstone}" :style="{opacity: +!item.loaded}">
          <slot name="tombstone"></slot>
        </div>
        <div :class="{'vue-recyclist-transition': tombstone}" :style="{opacity: +item.loaded}">
          <slot name="item" :data="item.data" :index="index"></slot>
        </div>
      </div>

      <!--get tombstone and item heights from these invisible doms-->
      <div class="vue-recyclist-pool">
        <div :ref="'item'+index" v-for="(item, index) in items" v-if="!item.tomb && !item.height"
          class="vue-recyclist-item vue-recyclist-invisible">
          <slot name="item" :data="item.data"></slot>
        </div>
        <div ref="tomb" class="vue-recyclist-item vue-recyclist-invisible">
          <slot name="tombstone"></slot>
        </div>
      </div>
    </div>

    <div v-show="spinner && !nomore && !tombstone"
      class="vue-recyclist-loading"
      :style="{visibility: loading ? 'visible' : 'hidden'}">
      <slot name="spinner">
        <div class="vue-recyclist-loading-content">
          <div class="cssloading-circle spinner"></div>
        </div>
      </slot>
    </div>

    <div v-show="nomore && !loading"
      class="vue-recyclist-nomore">
      <slot name="nomore">
        <div>End of list</div>
      </slot>
    </div>
  </div>
</template>
<script>
  export default {
    data() {
      return {
        name: 'VueRecyclist',
        // 包装后的所有的item，
        items: [], // Wrapped full list items
        // 容器内所有item的高度
        height: 0, // Full list height
        // 在有占位符的情况下，即tombstone为true的情况下，会同时加载多页，所以需要loading时个数组，表示多页的加载
        loadings: [], // Loading status queue
        // 可见的item的开始的index
        start: 0, // Visible items start index
        // start可见的item的offset
        startOffset: 0 // Start item offset
      }
    },
    computed: {
      // 真正在页面上展示的的items，即可见的items
      visibleItems() {
        return this.items.slice(Math.max(0, this.start - this.size), Math.min(this.items.length, this.start + this.size))
      },
      containerHeight() {
        return this.$el && this.$el.offsetHeight || 0
      },
      tombHeight() {
        return this.tombstone ? this.$refs.tomb && this.$refs.tomb.offsetHeight : 0
      },
      // 只要在加载中，就显示加载的模块
      loading() {
        return this.loadings.length
      }
    },
    props: {
      // 数据
      list: {
        type: Array,
        required: true
      },
      // 是否开启占位符
      tombstone: {
        type: Boolean,
        default: false // Whether to show tombstones.
      },
      // 每一页的大小，注意这里一页面的大小的高度一定要高于容器的高度
      size: {
        type: Number,
        default: 20 // The number of items added each time.
      },
      //上拉加载时候控制加载的
      offset: {
        type: Number,
        default: 100 // The number of pixels of additional length to allow scrolling to.
      },
      // loadingmore的函数，即加载下一页面的函数
      loadmore: {
        type: Function,
        required: true // The function of loading more items.
      },
      // 是否显示 loading的控件
      spinner: {
        type: Boolean,
        default: true // Whether to show loading spinner.
      },
      // 是否显示没有数据的控件
      nomore: {
        type: Boolean,
        default: false // Whether to show 'no more data' status bar
      }
    },
    watch: {
      // list长度发生变化后，如果是首次长度，就初始化，否则就加载更多的item，这个时候回将loading的状态弹出数组，表示一页已经加载好了
      list(arr) {
        if (arr.length) {
          this.loadings.pop()
          if (!this.loading) {
            this.loadItems()
          }
        } else {
          this.init()
        }
      },
      items(arr) {
        if (arr.length > this.list.length) {
          this.getItems()
        }
      }
    },
    mounted() {
      this.$el.addEventListener('scroll', this.onScroll.bind(this))
      window.addEventListener('resize', this.onResize.bind(this))
      this.init()
    },
    methods: {
      init() {
        this.reset()
        this.load()
      },
      reset() {
        this.items = []
        this.height = this.top = this.start = 0
        this.$el.scrollTop = 0
      },
      // 这里的load是指在有占位符的情况下，会有限直接展示，占位符，然后在进行数据的加载，否则就是直接加载数据
      load() {
        if (this.tombstone) {
          this.items.length += this.size
          // 并没有加载数据，去渲染 占位符，占位符渲染后,根据items的长度的变化，在去真正加载数据
          this.loadItems()
        } else if (!this.loading) {
          // 真正的调用加载数据的方法
          this.getItems()
        }
      },
      // 真正ajax加载下一页面
      getItems() {
        this.loadings.push(1)
        this.loadmore()
      },
      // 将数据拿回，然后先放到一个渲染队列，渲染后得到真实的每个item的高度，然后根据高度计算每个item的top值，和容器的总高度
      loadItems() {
        let loads = []
        let start = 0
        let end = this.tombstone ? this.items.length : this.list.length
        for (let i = start; i < end; i++) {
          if (this.items[i] && this.items[i].loaded) {
            continue
          }
          this.setItem(i, this.list[i] || null)
          // update newly added items position
          loads.push(this.$nextTick().then(() => {
            this.updateItemHeight(i)
          }))
        }
        // update items top and full list height
        Promise.all(loads).then(() => {
          this.updateItemTop()
        })
      },
      setItem(index, data) {
        this.$set(this.items, index, {
          data: data ? data : {},
          height: 0,
          top: -1000,
          tomb: !data,
          loaded: !!data
        })
      },
      updateItemHeight(index) {
        // update item height
        let cur = this.items[index]
        let dom = this.$refs['item' + index]
        // 如果存在dom就更新高度,否则就采用占位符的高度
        if (dom && dom[0]) {
          cur.height = dom[0].offsetHeight
        } else {
          // item is tombstone
          cur.height = this.tombHeight
        }
      },
      updateItemTop() {
        // loop all items to update item top and list height
        this.height = 0
        for (let i = 0; i < this.items.length; i++) {
          let pre = this.items[i - 1]
          this.items[i].top = pre ? pre.top + pre.height : 0
          this.height += this.items[i].height
        }
        // update scroll top when needed
        if (this.startOffset) {
          this.setScrollTop()
        }
        this.updateIndex()
        this.makeScrollable()
      },
      updateIndex() {
        // update visible items start index
        let top = this.$el.scrollTop
        for (let i = 0; i < this.items.length; i++) {
          if (this.items[i].top > top) {
            this.start = Math.max(0, i - 1)
            break
          }
        }
        // scrolling does not need recalculate scrolltop
        // this.getStartItemOffset()
      },
      getStartItemOffset() {
        if (this.items[this.start]) {
          this.startOffset = this.items[this.start].top - this.$el.scrollTop
        }
      },
      setScrollTop() {
        if (this.items[this.start]) {
          this.$el.scrollTop = this.items[this.start].top - this.startOffset
          // reset start item offset
          this.startOffset = 0
        }
      },
      makeScrollable() {
        // make ios -webkit-overflow-scrolling scrollable
        this.$el.classList.add('vue-recyclist-scrollable')
      },
      onScroll() {
        if (this.$el.scrollTop + this.$el.offsetHeight > this.height - this.offset) {
          this.load()
        }
        this.updateIndex()
      },
      onResize() {
        this.getStartItemOffset()
        this.items.forEach((item) => {
          item.loaded = false
        })
        this.loadItems()
      }
    },
    destroyed() {
      this.$el.removeEventListener('scroll', this.onScroll.bind(this))
      window.removeEventListener('resize', this.onResize.bind(this))
    }
  }

</script>
<style src="./cssloading.css"></style>
<style lang="scss" scoped>
  $duration: 500ms;
  .vue-recyclist {
    overflow-x: hidden;
    overflow-y: auto;
    &.vue-recyclist-scrollable {
      -webkit-overflow-scrolling: touch;
    }
    .vue-recyclist-items {
      position: relative;
      margin: 0;
      padding: 0;
      .vue-recyclist-invisible {
        top: -1000px;
        visibility: hidden;
      }
      .vue-recyclist-item {
        position: absolute;
        width: 100%;
        .vue-recyclist-transition {
          position: absolute;
          opacity: 0;
          transition-property: opacity;
          transition-duration: $duration;
        }
      }
    }
    .vue-recyclist-loading {
      overflow: hidden;
      .vue-recyclist-loading-content {
        width: 100%;
        text-align: center;
        .spinner {
          margin: 10px auto;
          width: 20px;
          height: 20px;
        }
      }
    }
    .vue-recyclist-nomore {
      overflow: hidden;
      margin: 10px auto;
      height: 20px;
      text-align: center;
    }
  }
</style>