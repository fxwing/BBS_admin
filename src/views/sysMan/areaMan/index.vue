<template>
  <div class="app-container">
    <!-- 左侧树 -->
    <div class="left">
      <page-tree
        :baseData.sync="treeInfo.baseData"
        :nodeKey="treeInfo.nodeKey"
        :lazy="true"
        :lazyInfo="treeInfo.lazyInfo"
        :rightMenuList="treeInfo.rightMenuList"
        :treeRefresh="treeInfo.refresh"
        :refreshLevel="treeInfo.refreshLevel"
        @handleClickBt="handleClickBt"
        @handleEvent="handleEvent"></page-tree>
    </div>
    <div class="right">
      <!-- 表格 -->
      <page-table
        :refresh="tableInfo.refresh"
        :initCurpage="tableInfo.initCurpage"
        :data.sync="tableInfo.data"
        :api="getListApi"
        :query="filterInfo.query"
        :fieldList="tableInfo.fieldList"
        :listTypeInfo="listTypeInfo"
        :handle="tableInfo.handle"
        @handleClickBt="handleClickBt"
        @handleEvent="handleEvent">
        <!-- 自定义插槽显示状态 -->
        <template v-slot:status="scope">
          <i
            :class="scope.row.status === 1 ? 'el-icon-check' : 'el-icon-close'"
            :style="{color: scope.row.status === 1 ? '#67c23a' : '#f56c6c', fontSize: '20px'}">
          </i>
        </template>
        <!-- 自定义插槽状态按钮 -->
        <template v-slot:bt_status="scope">
          <el-button
            v-if="scope.data.item.show && (!scope.data.item.ifRender || scope.data.item.ifRender(scope.data.row))"
            size="mini"
            :type="scope.data.row.status - 1 >= 0 ? 'danger' : 'success'"
            :icon="scope.data.item.icon"
            v-waves
            @click="handleClickBt(scope.data.item.event, scope.data.row)"
            :disabled="scope.data.item.disabled"
            :loading="scope.data.row[scope.data.item.loading]">
            {{scope.data.row.status - 1 >= 0 ? '停用' : '启用'}}
          </el-button>
        </template>
      </page-table>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import { getListApi, updateApi, getAllApi } from '@/api/sysMan/areaMan'
import HandleApi from '@/common/mixin/handleApi'
import PageTree from '@/components/PageTree'
import PageFilter from '@/components/PageFilter'
import PageTable from '@/components/PageTable'

export default {
  mixins: [HandleApi],
  components: {
    PageTree,
    PageFilter,
    PageTable
  },
  data () {
    return {
      getListApi,
      updateApi,
      getAllApi,
      // 相关列表
      listTypeInfo: {
        statusList: [
          {key: '启用', value: 1},
          {key: '停用', value: 0}
        ]
      },
      // 树相关信息
      treeInfo: {
        refresh: 1,
        refreshLevel: 0,
        nodeKey: 'key',
        lazy: true,
        type: 0, // 省市区类型
        baseData: [], // 树的基础数据，从组件中获取到
        lazyInfo: [
          {
            key: 'id', // 节点唯一标识字段
            label: 'name', // 节点显示字段
            type: 1, // 数据类型
            api: getAllApi, // 获取数据的接口
            params: {key: 'pid', value: 1, type: 'url'} // 获取数据的参数
          },
          {
            key: 'id',
            label: 'name',
            type: 2,
            api: getAllApi,
            params: {key: 'pid', value: '', type: 'url'},
            leaf: true
          }
        ],
        rightMenuList: []
      },
      // 过滤相关配置
      filterInfo: {
        query: {
          pid: 1
        }
      },
      // 表格相关
      tableInfo: {
        refresh: 1,
        initCurpage: 1,
        data: [],
        fieldList: [
          {label: '名称', value: 'name'},
          {label: '级别', value: 'level'},
          {label: '状态', value: 'status', type: 'slot', width: 80}
        ],
        handle: {
          fixed: 'right',
          label: '操作',
          width: '100',
          btList: [
            {label: '启用', type: 'success', icon: 'el-icon-process', event: 'status', loading: 'statusLoading', show: false, slot: true}
          ]
        }
      }
    }
  },
  computed: {
    ...mapGetters([
      'userInfo',
      'dataPerms'
    ])
  },
  watch: {
    'tableInfo.data' (val) {
      this.setList(this.treeInfo.type)
    }
  },
  created () {
    this.initParams()
  },
  mounted () {
    this.getList()
    this.initDataPerms()
  },
  methods: {
    // 初始化数据权限
    initDataPerms () {
      const btList = this.tableInfo.handle.btList
      for (let item of btList) {
        if (this.dataPerms.includes('areaMan:' + item.event)) {
          item.show = true
        }
      }
    },
    initParams () {
    },
    // 获取列表
    getList () {
      this.tableInfo.refresh = Math.random()
    },
    // 按钮点击
    handleClickBt (event, data) {
      switch (event) {
      case 'status':
        let params = JSON.parse(JSON.stringify(data))
        params.status = params.status - 1 >= 0 ? 0 : 1
        data.statusLoading = true
        delete params.statusLoading
        this._handleAPI('update', updateApi, params).then(res => {
          data.statusLoading = false
          if (res.success) {
            data.status = params.status
          }
        }).catch(() => {
          data.statusLoading = false
        })
        break
      }
    },
    // 触发事件
    handleEvent (event, data) {
      const tableInfo = this.tableInfo,
        treeInfo = this.treeInfo,
        filterInfo = this.filterInfo
      switch (event) {
      // 对表格获取到的数据做处理
      case 'list':
        if (!data) return
        // 初始化数据
        data.forEach(item => {
          item.statusLoading = false
        })
        break
      // 左键点击的处理
      case 'leftClick':
        treeInfo.type = data.data.type + 1
        filterInfo.query.pid = data.data.id
        // 重置分页
        tableInfo.initCurpage = Math.random()
        tableInfo.refresh = Math.random()
        break
      // 根据右键点击创建节点对应菜单
      case 'rightClick':
        treeInfo.rightMenuList = [
          {name: '刷新', type: 'refreshTree', data: null, node: null, vm: null}
        ]
        break
      // 右键菜单对应的事件处理
      case 'rightEvent':
        this.handleRightEvent(data.type, data)
        break
      }
    },
    // 具体的右键事务处理
    handleRightEvent (type, data) {
      const tableInfo = this.tableInfo,
        treeInfo = this.treeInfo,
        filterInfo = this.filterInfo
      switch (type) {
      case 'refreshTree':
        // 表格初始化
        filterInfo.query.pid = 1
        tableInfo.initCurpage = Math.random()
        tableInfo.refresh = Math.random()
        // falls through 告诉ESlint不检查这一行
      case 'refresh':
        // 树刷新
        treeInfo.refreshLevel = !data.node ? 0 : data.node.level
        treeInfo.refresh = Math.random()
        break
      }
    },
    setList (type) {
      let level = ''
      switch (type) {
      case 0:
        level = '省'
        break
      case 1:
        level = '省'
        break
      case 2:
        level = '市'
        break
      case 3:
        level = '区'
        break
      }
      this.tableInfo.data.forEach(item => {
        item.level = level
      })
    }
  }
}
</script>

<style scoped>

</style>
