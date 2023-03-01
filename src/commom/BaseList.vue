<template>
  <div class="app-container">

    <el-row v-if="!blueprint.pannel.hide" :gutter="40" class="panel-group">
      <div v-for="(item, index) in blueprint.pannel.fields" :key="index">
        <el-col :xs="12" :sm="12" :lg="6" class="card-panel-col">
          <div class="card-panel">
            <div class="card-panel-icon-wrapper icon-people">
              <svg-icon :icon-class="item.icon" class-name="card-panel-icon" />
            </div>
            <div class="card-panel-description">
              <div class="card-panel-text">
                {{ item.name }}
              </div>
              <span class="card-panel-num">{{ item.num }}</span>
            </div>
          </div>
        </el-col>
      </div>
    </el-row>

    <div class="filter-container">
      <div v-if="!blueprint.search.hide">
        <el-row>
          <el-col v-for="(field, index) in blueprint.search.fields" :key="index" :span="6">

            <el-input
              v-if="field.type === 'input'"
              v-model="search[field.key]"
              :placeholder="field.label"
              style="width: 200px;"
              class="filter-item"
              @keyup.enter.native="handleFilter"
            />

            <el-select
              v-else-if="field.type === 'select'"
              v-model="search[field.key]"
              :placeholder="field.label"
              clearable
              style="width: 200px"
              class="filter-item"
            >
              <el-option
                v-for="item in getOption(field.options)"
                :key="item.value"
                :label="item.text"
                :value="item.text"
              />
            </el-select>

            <el-date-picker
              v-else-if="field.type === 'date'"
              v-model="search[field.key]"
              type="datetime"
              :placeholder="field.label"
            />

          </el-col>
          <el-button v-waves class="filter-item" type="primary" icon="el-icon-search" @click="handleFilter">
            搜索
          </el-button>
        </el-row>
      </div>
      <hr v-if="!blueprint.search.hide && !blueprint.actions.hide">
      <div v-if="!blueprint.actions.hide">
        <el-row>
          <el-col v-for="(field, index) in blueprint.actions.right" :key="index" :span="6" :offset="20">
            <el-button
              v-if="field.type === 'add'"
              class="filter-item"
              style="margin-left: 10px;"
              type="primary"
              icon="el-icon-edit"
              @click="handleCreate"
            >
              {{ field.text }}
            </el-button>

            <el-button
              v-if="field.type === 'redirect'"
              class="filter-item"
              style="margin-left: 10px;"
              :type="field.variant || 'info'"
              @click="handleRedirect"
            >
              {{ field.text }}
            </el-button>

          </el-col>
        </el-row>
      </div>
    </div>

    <el-table v-loading="listLoading" :data="list" style="width: 100%">
      <div v-for="(item, index) in blueprint.table.fields" :key="index">
        {<el-table-column :prop="item.key" :label="item.label" :sortable="item.sortable" />}
      </div>
      <template v-if="blueprint.table.actions != ''">
        {{ blueprint.table.actions }}
        <el-table-column label="操作" align="center" width="230" class-name="small-padding fixed-width">
          <template slot-scope="{row,$index}">
            <div v-for="(field, index) in blueprint.table.actions" :key="index">
              <el-button
                v-if="field.type === 'edit'"
                type="primary"
                size="mini"
                @click="handleUpdate(row)"
              >
                修改
              </el-button>
              <el-button
                v-if="field.type === 'delete'"
                size="mini"
                type="danger"
                @click="handleDelete(row, $index)"
              >
                删除
              </el-button>
              <el-button
                v-if="field.type === 'custom'"
                :type="field.variant || 'info'"
                size="mini"
                @click="field.click(row)"
              >
                {{ field.text }}
              </el-button>
            </div>
          </template>
        </el-table-column>
      </template>

    </el-table>

    <pagination
      v-show="total > 0"
      :total="total"
      :page.sync="listQuery.page"
      :limit.sync="listQuery.limit"
      @pagination="getList"
    />

    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible">
      <el-form ref="dataForm" label-position="left" label-width="70px" style="width: 400px; margin-left:50px;">
        <div v-for="(field, index) in blueprint.form.fields" :key="index">

          <el-form-item v-if="field.type === 'input'" :label="field.label">
            <el-input v-model="formData[field.key]" />
          </el-form-item>

          <el-form-item v-if="field.type === 'redio'" :label="field.label">
            <el-radio-group v-model="formData[field.key]" size="medium">
              <el-radio-button
                v-for="item in getOption(field.options)"
                :key="item.value"
                :label="item.value"
                :value="item.value"
              >
                {{ item.text }}
              </el-radio-button>
            </el-radio-group>
          </el-form-item>

          <el-form-item v-if="field.type === 'select'" :label="field.label">
            <el-select
              v-model="formData[field.key]"
              :multiple="field.multiple"
              :filterable="field.filterable"
            >
              <el-option
                v-for="item in getOption(field.options)"
                :key="item.value"
                :label="item.text"
                :value="item.value"
              />
            </el-select>
          </el-form-item>

          <el-form-item v-if="field.type === 'img'" :label="field.label">
            <el-upload
              class="upload-demo"
              drag
              :action="field.actions"
              :multiple="field.multiple"
              :on-success="handleAvatarSuccess"
            >
              <i class="el-icon-upload" />
              <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>

              <img v-if="formData.publicWelfareUrl" :src="formData.publicWelfareUrl" class="avatar">
            </el-upload>
          </el-form-item>

          <el-form-item v-if="field.type === 'textarea'" :label="field.label">
            <el-input v-model="formData[field.key]" type="textarea" />
          </el-form-item>

          <el-form-item v-if="field.type === 'cascader'" :label="field.label">
            <el-cascader v-model="formData[field.key]" :options="getOption(field.options)" @change="handleCascaderChange" />
          </el-form-item>
        </div>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">
          取消
        </el-button>
        <el-button type="primary" @click="dialogStatus === 'create' ? createData() : updateData()">
          确认
        </el-button>
      </div>
    </el-dialog>

  </div>
</template>

<script>
import waves from '@/directive/waves' // waves directive
import Pagination from '@/components/Pagination' // secondary package based on el-pagination
export default {
  components: { Pagination },
  directives: { waves },
  data() {
    return {
      checkList: [],
      search: {},
      formData: {},
      listLoading: false,
      dialogFormVisible: false,
      tableKey: 0,
      list: null,
      total: 0,
      dialogStatus: 'create',
      listQuery: {
        page: 1,
        limit: 20
      },
      textMap: {
        update: '修改',
        create: '新增'
      },
      blueprint: {
        actions: {
          hide: true,
          left: [],
          right: []
        },
        pannel: {
          hide: true,
          fields: []
        },
        search: {
          hide: true,
          params: {
            page: 1,
            orderBy: 'id',
            orderDir: 'desc'
          },
          fields: [],
          ext: function() {
            return {}
          }
        },
        table: {
          fields: [],
          actions: []
        },
        form: {
          fields: []
        }
      }
    }
  },
  created() {
    this.getList()
  },
  methods: {
    handleCascaderChange(value) {
      console.log(value)
    },
    handleAvatarSuccess(response, file, fileList) {
      console.log(response)
      if (response && response.data && response.data.url) {
        this.$set(this.formData, 'publicWelfareUrl', response.data)
      }
      this.formData.publicWelfareUrl = response.data
    },
    getList() { },
    handleFilter() {
      console.log(this.search)
    },
    sortChange() { },
    createData() { },
    updateData() { },
    handleDelete() { },
    handleRedirect() { },
    handleUpdate(row) {
      this.formData = row
      if (this.formData.password) {
        this.formData.password = ''
      }
      this.dialogStatus = 'update'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    handleCreate() {
      this.dialogStatus = 'create'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    getOption(option) {
      switch (typeof option) {
        case 'string':
          if (typeof this[option] === 'function') {
            return this[option]()
          } else {
            return this[option]
          }
        case 'function':
          return option()
        default:
          return option
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.avatar {
    width: 178px;
    height: 178px;
    display: block;
}

.panel-group {
    margin-top: 18px;

    .card-panel-col {
        margin-bottom: 32px;
    }

    .card-panel-button {
        margin-bottom: 32px;
        margin-left: 32px;
    }

    .card-panel {
        height: 108px;
        cursor: pointer;
        font-size: 12px;
        position: relative;
        overflow: hidden;
        color: #666;
        background: #fff;
        box-shadow: 4px 4px 40px rgba(0, 0, 0, .05);
        border-color: rgba(0, 0, 0, .05);

        &:hover {
            .card-panel-icon-wrapper {
                color: #fff;
            }

            .icon-people {
                background: #40c9c6;
            }

            .icon-message {
                background: #36a3f7;
            }

            .icon-money {
                background: #f4516c;
            }

            .icon-shopping {
                background: #34bfa3
            }
        }

        .icon-people {
            color: #40c9c6;
        }

        .icon-message {
            color: #36a3f7;
        }

        .icon-money {
            color: #f4516c;
        }

        .icon-shopping {
            color: #34bfa3
        }

        .card-panel-icon-wrapper {
            float: left;
            margin: 14px 0 0 14px;
            padding: 16px;
            transition: all 0.38s ease-out;
            border-radius: 6px;
        }

        .card-panel-icon {
            float: left;
            font-size: 48px;
        }

        .card-panel-description {
            float: right;
            font-weight: bold;
            margin: 26px;
            margin-left: 0px;

            .card-panel-text {
                line-height: 18px;
                color: rgba(0, 0, 0, 0.45);
                font-size: 16px;
                margin-bottom: 12px;
            }

            .card-panel-num {
                font-size: 20px;
            }
        }
    }
}
</style>
