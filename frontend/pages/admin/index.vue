<template>
  <div class="app-container">
    <div class="page-title">
      {{ $t('admin.list') }}
    </div>
    <el-card class="box-card search-box">
      <div class="top-filter-container">
        <search-filter-item
          :sel-preset="search.keyword_type"
          :search-fields="searchFields"
          :search-word="search.search_keyword"
          :option1="$t('common.include')"
          :option2="$t('common.match')"
          @searchWordChanged="searchWordChanged"
          @optionChanged="optionChanged"
          @handleSearch="handleSearch"
          @keywordTypeChanged="keywordTypeChanged"
        >
        </search-filter-item>
        <el-button class="btn-search" type="primary" icon="el-icon-search" @click="handleFilter">
          {{ $t('common.search') }}
        </el-button>
      </div>
    </el-card>
    <el-card class="box-card table-card">
      <label class="total-count">
        {{ $t('table.total_count', { total: total }) }}
      </label>
      <div class="table-header">
        <i class="el-icon-delete" @click="delSelAdmin"></i>
        <div class="table-header-actions">
          <el-button class="filter-item btn-add" type="primary" icon="el-icon-plus" @click="goToCreatePage">
            {{ $t('requester.new_account') }}
          </el-button>
          <el-button :loading="downloadLoading" class="btn-excel-download" type="primary" icon="el-icon-download" @click="handleDownload">
            {{ $t('common.excel_download') }}
          </el-button>
          <el-dropdown trigger="click">
            <span class="el-dropdown-link">
              <el-button class="filter-item btn-add" icon="el-icon-s-tools">
              </el-button>
            </span>
            <el-dropdown-menu slot="dropdown" class="column-control-dropdown">
              <div class="column-control-desc">
                {{ $t('common.column_control_desc') }}
              </div>
              <el-checkbox v-model="checkAllColumns" class="mt10" :indeterminate="isIndeterminate" @change="handleCheckAllColumnChange">
                {{ $t('common.all') }}
              </el-checkbox>
              <el-checkbox-group v-model="columnCheckVal" @change="handleCheckedColumnChange">
                <draggable
                  v-model="totalColumns"
                  class="list-group"
                  ghost-class="ghost"
                  @start="dragging = true"
                  @end="dragging = false"
                >
                  <el-checkbox v-for="(item, i) in totalColumns" :key="i" :label="item">
                    {{ $t('table.' + item) }}
                  </el-checkbox>
                </draggable>
              </el-checkbox-group>
            </el-dropdown-menu>
          </el-dropdown>
        </div>
      </div>
      <el-table
        :key="tableKey"
        v-loading="listLoading"
        :data="list"
        fit
        border
        highlight-current-row
        style="width: 100%;"
        :row-class-name="tableRowClassName"
        @selection-change="handleSelectionChange"
      >
        <el-table-column type="selection" width="55" align="center"></el-table-column>
        <el-table-column
          :label="$t('table.no')"
          type="index"
          width="70"
          align="center"
          :index="indexMethod"
        >
        </el-table-column>
        <el-table-column :label="$t('table.detail')" width="80" align="center">
          <template slot-scope="{row}">
            <el-button size="mini" icon="el-icon-document" class="btn-detail" @click="viewDetail(row)">
            </el-button>
          </template>
        </el-table-column>
        <el-table-column
          v-for="item in tableColumnOptions"
          :key="item"
          :label="$t('table.' + item)"
          align="center"
        >
          <template slot-scope="scope">
            <span v-if="item=='reg_date'">{{ scope.row[item] | date }}</span>
            <span v-else>{{ scope.row[item] ? scope.row[item] : '-' }}</span>
          </template>
        </el-table-column>
      </el-table>

      <pagination v-show="total > 0" :total="total" :page.sync="table.page" :limit.sync="table.page_length" @pagination="getList" />
    </el-card>
  </div>
</template>

<script>
import list from '@/mixin/list'
// eslint-disable-next-line no-unused-vars
import * as exportExcel from '@/utils/Export2Excel'

const defaultColumns = [
  'user_id',
  'user_name',
  'phone_number',
  'email',
  'reg_date'
]

export default {
  middleware: ['auth', 'auth-admin'],
  name: 'AdminList',
  mixins: [list],
  data () {
    return {
      table: {
        page: 1,
        page_length: 10
      },
      search: {
        keyword_type: 1,
        search_keyword: '',
        search_type: 1 // 1: 포함, 2: 일치
      },
      searchFields: [
        { value: 1, label: this.$t('table.user_name') },
        { value: 2, label: this.$t('table.user_id') },
        { value: 3, label: this.$t('table.phone_number') },
        { value: 4, label: this.$t('table.email') }
      ]
    }
  },
  created () {
    /** Call api */
    this.initTableColumnData()
    this.getList()
  },
  methods: {
    initTableColumnData () {
      this.totalColumns = defaultColumns
      this.tableColumnOptions = defaultColumns
      this.columnCheckVal = defaultColumns
    },
    goToCreatePage () {
      this.$router.push('/admin/add')
    },
    indexMethod (index) {
      return this.total - ((this.table.page - 1) * this.table.page_length + index) // descending order
    },
    // For custom class names
    tableRowClassName ({ row, rowIndex }) {
      if (rowIndex % 2 === 0) {
        return 'even-row'
      } else {
        return 'odd-row'
      }
    },
    handleSelectionChange (val) {
      this.multipleSelection = val
    },
    delSelAdmin () {
      if (this.multipleSelection.length === 0) {
        this.$alert(this.$t('validation.select_del_admin'), this.$t('validation.delete'), {
          confirmButtonText: 'OK'
        })
      } else {
        this.$confirm(this.$t('validation.delete_admin'), this.$t('validation.delete'), {
          confirmButtonText: 'OK',
          cancelButtonText: 'Cancel'
        }).then(async () => {
          this.listLoading = true
          const idList = []
          const delCondition = true
          this.multipleSelection.map((select) => {
            idList.push(select.id)
          })
          if (delCondition) {
            const response = await this.$axios.post('/admin/delete-admin', {
              ids: idList
            })
            if (response.data.errorCode === 0) {
              this.$message({
                type: 'success',
                message: this.$t('validation.delete_success')
              })
              this.getList()
            } else {
              this.$message({
                type: 'warning',
                message: this.$t('validation.delete_fail')
              })
              this.listLoading = false
            }
          } else {
            this.$message({
              type: 'warning',
              message: this.$t('validation.delete_no_admin')
            })
            this.listLoading = false
          }
        }).catch(() => {
        })
      }
    },
    async getList () {
      this.listLoading = true
      // eslint-disable-next-line camelcase
      const { page, page_length } = this.table
      const searchParams = Object.assign({}, this.search)
      try {
        const response = await this.$axios.post('/admin/get-admin-list', {
          table: { page, page_length },
          search: searchParams
        })
        this.list = response.data.data.list
        this.total = response.data.data.totalCount
      } catch (err) {
        console.log(err)
      } finally {
        this.listLoading = false
      }
    },
    handleFilter () {
      this.table.page = 1
      this.getList()
    },
    optionChanged (option) {
      this.search.search_type = option
    },
    handleSearch () {
      this.table.page = 1
      this.getList()
    },
    keywordTypeChanged (keywordType) {
      this.search.keyword_type = keywordType
    },
    searchWordChanged (searchWord) {
      this.search.search_keyword = searchWord
    },
    viewDetail (row) {
      const popupSize = 'width=' + this.variables.popupWidth + ',height=' + this.variables.popupHeight
      window.open('/admin/detail/' + row.id, '_blank', popupSize)
    },
    handleDelete (row, index) {
      this.$notify({
        title: 'Success',
        message: 'Delete Successfully',
        type: 'success',
        duration: 2000
      })
      this.list.splice(index, 1)
    },
    async handleDownload () {
      const tHeader = [this.$t('table.no')]
      const filterVal = ['no']
      for (let i = 0; i < this.tableColumnOptions.length; i++) {
        tHeader.push(this.$t('table.' + this.tableColumnOptions[i]))
        filterVal.push(this.tableColumnOptions[i])
      }
      this.downloadLoading = true
      const data = await this.formatJson(filterVal)
      exportExcel.exportJsonToExcel({
        header: tHeader,
        data,
        filename: 'list_of_admins'
      })
      this.downloadLoading = false
    },
    async formatJson (filterVal) {
      const searchParams = Object.assign({}, this.search)
      try {
        const response = await this.$axios.post('/admin/get-admin-list', {
          search: searchParams
        })
        this.excelList = response.data.data.list
      } catch (err) {
        this.excelList = []
      }
      return this.excelList.map((v, index) => filterVal.map((j) => {
        if (j === 'no') {
          return index + 1
        } else if (j === 'reg_date') {
          return this.$options.filters.date(v[j])
        } else {
          return v[j]
        }
      }))
    },
    numberWithCommas (x) {
      return x.toString().replace(/\B(?<!\.\d*)(?=(\d{3})+(?!\d))/g, ',')
    }
  }
}
</script>
<style lang="scss" scoped>
@import '@/assets/styles/variables.scss';
.column-control-desc {
  max-width: 300px;
}
.btn-detail {
  background: $blue10;
}

::v-deep {
  .el-input__prefix {
    left: 10px;
    top: 10px;
  }
  .btn-detail.el-button--mini {
    font-size: 14px;
    line-height: 2;
    padding: 4px 8px;
  }
  .el-icon-document:before {
    color: $blue50 !important;
  }
}
</style>
