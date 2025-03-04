<template>
  <div class="container">
    <div v-show="!focusMode" class="banner">
      <div class="banner-content">
        <h2>Banner 区域</h2>
        <n-button @click="toggleFocusMode">进入专注模式</n-button>
      </div>
    </div>
    <div v-show="focusMode" class="focus-toggle">
      <n-button @click="toggleFocusMode">退出专注模式</n-button>
    </div>
    <div class="table-container" :class="{ 'focus-mode': focusMode }">
      <n-data-table
        :columns="columns"
        :data="currentPageData"
        :pagination="pagination"
        :row-props="rowProps"
        :loading="loading"
        remote
        :bordered="false"
        :expanded-row-keys="expandedRowKeys"
        @update:expanded-row-keys="handleExpandChange"
        @update:page="handlePageChange"
        @update:page-size="handlePageSizeChange"
        ref="tableRef"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, nextTick, watch, onUnmounted, h } from 'vue'
import type { DataTableColumns } from 'naive-ui'

const focusMode = ref(false)
const tableData = ref<any[]>([])
const loading = ref(false)
const dataCache = ref<Map<number, any[]>>(new Map())
const totalItems = ref(0) // 替换固定的 TOTAL_ITEMS

const pagination = ref({
  page: 1,
  pageSize: 10,
  itemCount: 0, // 初始值设为 0
  showSizePicker: true,
  pageSizes: [10, 20, 30, 40],
  prefix: ({ itemCount }: { itemCount: number }) => `共 ${itemCount} 条数据`
})

// 示例数据列定义
const columns: DataTableColumns = [
  {
    type: 'expand',
    expandable: (rowData) => true,
    renderExpand: (rowData) => {
      return h('div', { class: 'expand-content' }, [
        h('p', `详细信息 - ${rowData.name}`),
        h('p', `更多内容...`)
      ])
    }
  },
  {
    title: '名称',
    key: 'name',
    width: 120
  },
  {
    title: '年龄',
    key: 'age',
    width: 80
  },
  {
    title: '邮箱',
    key: 'email',
    width: 200
  },
  {
    title: '电话',
    key: 'phone',
    width: 150
  },
  {
    title: '地址',
    key: 'address',
    width: 200
  }
]

// 记录展开状态
const expandedRowKeys = ref<string[]>([])

// 模拟异步数据获取
const fetchPageData = async (page: number, pageSize: number) => {
  // 如果缓存中已有数据，直接返回
  if (dataCache.value.has(page)) {
    return {
      data: dataCache.value.get(page),
      total: totalItems.value
    }
  }

  // 模拟网络请求延迟
  await new Promise(resolve => setTimeout(resolve, 500))
  
  // 模拟接口返回的数据结构
  const total = 1000 // 这里应该是实际接口返回的总数
  const start = (page - 1) * pageSize
  const end = Math.min(start + pageSize, total)
  const pageData = Array.from({ length: end - start }, (_, index) => ({
    key: start + index,
    name: `用户 ${start + index}`,
    age: Math.floor(Math.random() * 40) + 20,
    address: `地址 ${start + index}`,
    email: `user${start + index}@example.com`,
    phone: `1${Math.floor(Math.random() * 1000000000).toString().padStart(10, '0')}`,
  }))

  // 将数据存入缓存
  dataCache.value.set(page, pageData)
  
  // 更新总数量
  totalItems.value = total
  pagination.value.itemCount = total

  return {
    data: pageData,
    total
  }
}

// 预加载下一页数据
const preloadNextPage = async (currentPage: number) => {
  const totalPages = Math.ceil(totalItems.value / pagination.value.pageSize)
  const nextPage = currentPage + 1
  
  // 只在不是最后一页时预加载
  if (nextPage <= totalPages && !dataCache.value.has(nextPage)) {
    console.log('预加载第', nextPage, '页数据')
    const response = await fetchPageData(nextPage, pagination.value.pageSize)
    if (response.data) {
      dataCache.value.set(nextPage, response.data)
    }
  }
}

// 计算当前页数据
const currentPageData = computed(() => {
  const currentPage = pagination.value.page
  const data = dataCache.value.get(currentPage)
  return data || []
})

// 初始化数据
const initData = async () => {
  loading.value = true
  try {
    const response = await fetchPageData(1, pagination.value.pageSize)
    if (response.data) {
      dataCache.value.set(1, response.data)
      // 确保设置第一页为当前页
      pagination.value = {
        ...pagination.value,
        page: 1,
        itemCount: response.total
      }
    }
    await preloadNextPage(1)
  } finally {
    loading.value = false
  }
}

// 处理页面切换
const handlePageChange = async (page: number) => {
  const totalPages = Math.ceil(totalItems.value / pagination.value.pageSize)
  if (page < 1 || page > totalPages) return

  loading.value = true
  pagination.value.page = page
  expandedRowKeys.value = [] // 清空展开状态
  
  try {
    const response = await fetchPageData(page, pagination.value.pageSize)
    if (response.data) {
      dataCache.value.set(page, response.data)
    }
    await preloadNextPage(page)
  } finally {
    loading.value = false
  }
}

// 处理每页条数变化
const handlePageSizeChange = async (pageSize: number) => {
  loading.value = true
  const totalPages = Math.ceil(totalItems.value / pageSize)
  
  pagination.value = {
    ...pagination.value,
    pageSize,
    page: 1,
    itemCount: totalItems.value
  }
  
  dataCache.value.clear() // 清空缓存
  
  try {
    const response = await fetchPageData(1, pageSize)
    if (response.data) {
      dataCache.value.set(1, response.data)
    }
    await preloadNextPage(1)
  } finally {
    loading.value = false
  }
}

// 处理行展开状态变化
const handleExpandChange = (keys: string[]) => {
  expandedRowKeys.value = keys
}

// 固定行高 - 修改为考虑展开内容的高度
const rowProps = (row: any) => {
  return {
    style: `height: ${expandedRowKeys.value.includes(row.key) ? '148px' : '48px'};`, // 展开后高度为 48 + 100
    'data-key': row.key
  }
}

// 修改计算每页显示的行数
const calculatePageSize = async () => {
  await nextTick()
  
  const tableContainer = document.querySelector('.table-container')
  if (!tableContainer) return
  
  const containerHeight = focusMode.value ? window.innerHeight : window.innerHeight - 80
  const headerHeight = 50 // 表头高度
  const paginationHeight = 45 // 分页器高度
  const baseRowHeight = 48 // 基础行高
  
  const availableHeight = containerHeight - headerHeight - paginationHeight
  const newPageSize = Math.max(1, Math.floor(availableHeight / baseRowHeight))
  
  // 如果页大小发生变化，需要清空缓存并重新加载数据
  if (pagination.value.pageSize !== newPageSize) {
    const currentPage = pagination.value.page
    const totalPages = Math.ceil(totalItems.value / newPageSize)
    
    // 确保页码不超出范围
    const validPage = Math.min(currentPage, totalPages)
    
    pagination.value = {
      ...pagination.value,
      page: validPage,
      pageSize: newPageSize,
      itemCount: totalItems.value,
      showSizePicker: true,
      pageSizes: [10, 20, 30, 40],
      prefix: ({ itemCount }: { itemCount: number }) => `共 ${itemCount} 条数据`
    }
    
    dataCache.value.clear()
    const response = await fetchPageData(validPage, newPageSize)
    if (response.data) {
      dataCache.value.set(validPage, response.data)
    }
  }

  console.log('当前模式:', focusMode.value ? '专注模式' : '普通模式')
  console.log('容器高度:', containerHeight)
  console.log('可用高度:', availableHeight)
  console.log('计算得到的每页条数:', newPageSize)
}

const scrollAccumulator = ref(0)
const FIRST_SCROLL_THRESHOLD = 600; // 增加第一次滚动需要的阈值
const SECOND_SCROLL_THRESHOLD = 100; // 增加第二次滚动需要的阈值
const isNextPageHighlighted = ref(false)
const tableRef = ref<HTMLElement | null>(null)

let isThrottled = false; // 节流标志

// 监听表格挂载
const initWheelEvent = () => {
  // 给一点时间让表格完全渲染
  setTimeout(() => {
    const tableContainer = document.querySelector('.table-container')
    console.log('表格容器元素:', tableContainer)
    
    if (tableContainer) {
      // 移除旧的事件监听
      tableContainer.removeEventListener('wheel', handleWheel as EventListener)
      // 添加新的事件监听
      tableContainer.addEventListener('wheel', handleWheel as EventListener, { passive: false })
      console.log('已添加滚轮事件监听')
    } else {
      console.log('未找到表格容器元素')
    }
  }, 200)
}

// 在组件卸载时移除事件监听
onUnmounted(() => {
  window.removeEventListener('resize', calculatePageSize)
  const tableContainer = document.querySelector('.table-container')
  if (tableContainer) {
    tableContainer.removeEventListener('wheel', handleWheel as EventListener)
  }
})

// 监听专注模式变化时重置累加器
watch(focusMode, () => {
  if (focusMode.value) {
    scrollAccumulator.value = 0
    initWheelEvent()
  } else {
    const tableContainer = document.querySelector('.table-container')
    if (tableContainer) {
      tableContainer.removeEventListener('wheel', handleWheel as EventListener)
    }
  }
})

// 高亮下一页页码
const highlightNextPage = () => {
  const nextPage = pagination.value.page + 1
  console.log('尝试高亮页码:', nextPage)
  
  // 延迟一下等待分页器渲染
  setTimeout(() => {
    const paginationItems = document.querySelectorAll('.n-data-table .n-pagination-item')
    console.log('找到分页器项数量:', paginationItems.length)
    
    paginationItems.forEach(item => {
      console.log('分页器项内容:', item.textContent)
      if (item.textContent?.trim() === String(nextPage)) {
        item.classList.add('highlight-next-page')
        console.log('成功添加高亮到页码:', nextPage)
      }
    })
  }, 0)
}

// 移除高亮
const removeHighlight = () => {
  console.log('尝试移除高亮')
  document.querySelectorAll('.n-data-table .n-pagination-item.highlight-next-page').forEach(el => {
    el.classList.remove('highlight-next-page')
    console.log('已移除高亮')
  })
}

// 处理鼠标滚轮事件
const handleWheel = (event: Event) => {
  if (!focusMode.value) return;
  console.log('收到滚轮事件');

  const wheelEvent = event as WheelEvent;
  // 阻止默认滚动
  wheelEvent.preventDefault();

  // 添加节流，防止快速滚动
  if (loading.value || isThrottled) {
    console.log('正在加载中或节流中，忽略滚动');
    return;
  }

  isThrottled = true; // 设置节流标志
  setTimeout(() => {
    isThrottled = false; // 重置节流标志
  }, 200); // 200ms 的节流时间

  // 使用 setTimeout 来延迟执行翻页逻辑
  setTimeout(() => {
    if (wheelEvent.deltaY > 0) {
      console.log('向下滚动, deltaY:', wheelEvent.deltaY);
      if (!isNextPageHighlighted.value) {
        // 第一次向下滚动，高亮下一页
        const nextPage = pagination.value.page + 1;
        const totalPages = Math.ceil(totalItems.value / pagination.value.pageSize);
        if (nextPage <= totalPages) {
          isNextPageHighlighted.value = true;
          highlightNextPage();
          console.log('已设置高亮状态');
        }
      } else {
        // 第二次向下滚动，执行翻页
        isNextPageHighlighted.value = false;
        removeHighlight();
        const nextPage = pagination.value.page + 1;
        const totalPages = Math.ceil(totalItems.value / pagination.value.pageSize);
        if (nextPage <= totalPages) {
          console.log('执行翻页到:', nextPage);
          handlePageChange(nextPage);
        }
      }
    } else if (wheelEvent.deltaY < 0) {
      console.log('向上滚动, deltaY:', wheelEvent.deltaY);
      if (!isNextPageHighlighted.value) {
        // 第一次向上滚动，高亮上一页
        const prevPage = pagination.value.page - 1;
        if (prevPage >= 1) {
          isNextPageHighlighted.value = true;
          highlightPrevPage();
          console.log('已设置上一页高亮状态');
        }
      } else {
        // 第二次向上滚动，执行翻页
        isNextPageHighlighted.value = false;
        removeHighlight();
        const prevPage = pagination.value.page - 1;
        if (prevPage >= 1) {
          console.log('返回上一页:', prevPage);
          handlePageChange(prevPage);
        }
      }
    }
  }, 0);
};

// 高亮上一页页码
const highlightPrevPage = () => {
  const prevPage = pagination.value.page - 1
  console.log('尝试高亮上一页页码:', prevPage)
  
  // 延迟一下等待分页器渲染
  setTimeout(() => {
    const paginationItems = document.querySelectorAll('.n-data-table .n-pagination-item')
    console.log('找到分页器项数量:', paginationItems.length)
    
    paginationItems.forEach(item => {
      console.log('分页器项内容:', item.textContent)
      if (item.textContent?.trim() === String(prevPage)) {
        item.classList.add('highlight-next-page')
        console.log('成功添加高亮到页码:', prevPage)
      }
    })
  }, 0)
}

const toggleFocusMode = () => {
  focusMode.value = !focusMode.value
  calculatePageSize() // 切换模式时重置到第一页
}

// 处理窗口大小变化
const handleResize = (event: UIEvent) => {
  calculatePageSize() // 窗口大小变化时保持当前页
};

onMounted(async () => {
  await calculatePageSize()
  await initData() // 替换原来的 handlePageChange(1)
  
  // 监听窗口大小变化
  window.addEventListener('resize', handleResize)
})
</script>

<style scoped>
.container {
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.banner {
  background-color: #f5f5f5;
  padding: 16px;
  height: 80px;
}

.banner-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.focus-toggle {
  position: fixed;
  top: 16px;
  right: 16px;
  z-index: 100;
}

.table-container {
  flex: 1;
  height: calc(100vh - 80px);
  display: flex;
  flex-direction: column;
  transition: height 0.3s ease;
}

.n-data-table {
  flex: 1;
  overflow: hidden;
}

.table-container.focus-mode {
  height: 100vh;
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 99;
  background: #fff;
  overflow: hidden;
}

:deep(.highlight-next-page) {
  background-color: #18a058 !important;
  color: #fff !important;
  transition: all 0.3s ease;
  position: relative;
  z-index: 1;
}

:deep(.highlight-next-page)::after {
  content: '';
  position: absolute;
  top: -2px;
  left: -2px;
  right: -2px;
  bottom: -2px;
  border: 2px solid #18a058;
  border-radius: 4px;
  animation: pulse 1s infinite;
}

@keyframes pulse {
  0% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.1);
    opacity: 0.8;
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

:deep(.n-pagination-item) {
  transition: all 0.3s ease;
  position: relative;
}

:deep(.n-scrollbar) {
  overflow: hidden !important;
}

:deep(.n-scrollbar-rail) {
  display: none !important;
}

.expand-content {
  padding: 16px;
  background-color: #f9f9f9;
}

:deep(.n-data-table-td--expanded) {
  background-color: #f9f9f9;
}
</style>