<template>
  <van-swipe :height="imageList.length || imgLoading ? 150 : 0" lazy-render>
    <van-swipe-item v-if="imageList.length" v-for="_item in imageList" :key="_item.content_id">
      <img :src="`${imgUrl}${_item.ipfs ? _item.ipfs : _item.thumbnail_array[1]?.ipfs}`" @click="handleDetailJump(_item)" />
      <p class="ad-desc">{{ _item.name }}</p>
    </van-swipe-item>
    <template v-if="imageList.length" #indicator="{ active, total }">
      <div class="custom-indicator">{{ active + 1 }}/{{ total }}</div>
    </template>
  </van-swipe>
  <!-- <van-pull-refresh v-model="refreshing" class="list-wrapper"> -->
    <van-list
      v-model:loading="loading"
      :finished="finished"
      finished-text="加载完毕"
      @load="onLoad"
    >
      <article-item v-for="(item, index) in articleList" :key="index" :article-detail="item" />
    </van-list>
  <!-- </van-pull-refresh> -->
</template>

<script lang="ts" setup>
import apis from '@/api'
import ArticleItem from './ArticleItem.vue'
import { imgUrl } from '@/utils/const'
const props = defineProps({
  columnId: {
    type: String,
    require: true
  },
  shopId: {
    type: String,
    require: true
  }
})
// const shopId:any = inject<string>('shopId')
const articleList = ref([])
const imageList = ref([])
const imgLoading = ref(true)
const current = ref(1)
// const refreshing = ref(false)
const loading = ref(false)
const finished = ref(false)

// watch(() => props.columnId, () => {
//   getArticleList()
// })

// 获取栏目详情查找广告
const getColumnDetail = async () => {
  const { data } = await apis.get_cmscolumn_detail({
    id: props.columnId
  })
  if(!data?.column_layout) return
  const layout = JSON.parse(data.column_layout)?.layout || []
  // 找type为c2的中央轮播图
  const layoutOption = layout.find(item => item.type === 'c2')
  
  if(!layoutOption.component_id) {
    imgLoading.value = false
    return
  }
  const res = await apis.get_column_ad_list({
    component_id: layoutOption.component_id
  })
  imageList.value = res?.data || []
  imgLoading.value = false
}

// 文章列表
const getArticleList = () => {
  const params = {
    current: current.value,
    page_size: 8,
    es_query: true,
    order_bys: "publishing_at|desc",
    param: {
      article_type: 20,
      is_first: true,
      public_type: 1,
      select_type: 1,
      column_id: props.columnId,
      shop_id: props.shopId
    }
  }
  apis.get_article_list(params).then(({data, page}) => {
    // if (refreshing.value) {
    //   articleList.value = []
    //   refreshing.value = false
    //   return
    // }
    articleList.value.push(...data)
    loading.value = false
    // 数据全部加载完成
    if (articleList.value.length >= page.total) {
      finished.value = true
    } else {
      current.value++
    }
  })
}

getColumnDetail()

const handleDetailJump = (info: any) => {
  const { publishing_id, column_id, biz_id } = info
  let origin: string = ''
  if(process.env.NODE_ENV === 'development') {
    origin = process.env.VUE_ARTICLE_DETAIL_URL
  } else {
    origin = window.location.origin
  }
  window.location.href = `${origin}/srvContent/view?id=${publishing_id}&column_id=${column_id}&biz_id=${biz_id}&slug&qr_code&columnType=4&openFrom=h5`
}

// const onRefresh = () => {}

const onLoad = () => {
  getArticleList()
}
</script>

<style lang="less" scoped>
.van-swipe-item img {
  width: 100%;
}
.ad-desc {
  position: absolute;
  left: 15px;
  bottom: 25px;
  font-size: 14px;
  color: @icWhite;
}
.custom-indicator {
  position: absolute;
  right: 5px;
  top: 5px;
  padding: 2px 5px;
  font-size: 12px;
  background: rgba(0, 0, 0, 0.1);
}
.list-wrapper {
  margin-top: 5px;
}
</style>
