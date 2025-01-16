<template>
  <div class="editable-table">
    <el-table :data="innerTableData" border :span-method="handleSpanMethod">
      <!-- 自定义序号列 -->
      <el-table-column
        label="序号"
        width="60"
        align="center"
      >
        <template slot-scope="scope">
          <!-- 新增行显示 "新增" -->
          <span v-if="scope.row.isNew">新增</span>
          <!-- 其他行显示序号 -->
          <span v-else>{{ scope.$index }}</span>
        </template>
      </el-table-column>

      <!-- 动态生成表格列 -->
      <el-table-column
        v-for="col in columns"
        :key="col.prop"
        v-bind="col"
      >
        <template slot-scope="scope">
          <!-- 编辑状态下显示输入框 -->
          <template v-if="scope.row.isEditing || scope.row.isNew">
            <!-- 下拉选择框 -->
            <el-select
              v-if="col.type === 'select'"
              v-model="scope.row[col.prop]"
              :disabled="isFieldDisabled(scope.row, col.prop)"
              @change="handleSelectChange(scope.row, col.prop)"
            >
              <el-option
                v-for="option in col.options"
                :key="option.value"
                :label="option.label"
                :value="option.value"
              />
            </el-select>
            <!-- 带按钮的输入框 -->
            <template v-else-if="col.type === 'input-button'">
              <div class="input-button-wrapper">
                <el-input
                  v-model="scope.row[col.prop]"
                  :disabled="isFieldDisabled(scope.row, col.prop)"
                  @change="handleSelectChange(scope.row, col.prop)"
                  style="width: calc(100% - 80px)"
                />
                <el-button
                  type="primary"
                  size="small"
                  :disabled="isFieldDisabled(scope.row, col.prop)"
                  @click="handleButtonClick(scope.row, col)"
                >
                  {{ col.buttonText || '获取' }}
                </el-button>
              </div>
            </template>
            <!-- 普通输入框 -->
            <el-input
              v-else
              v-model="scope.row[col.prop]"
              :disabled="isFieldDisabled(scope.row, col.prop)"
              @change="handleSelectChange(scope.row, col.prop)"
            />
          </template>
          <!-- 非编辑状态下显示文本 -->
          <span v-else>{{ scope.row[col.prop] }}</span>
        </template>
      </el-table-column>

      <!-- 操作列 -->
      <el-table-column label="操作" width="280" fixed="right">
        <template slot-scope="scope">
          <!-- 新行的操作按钮 -->
          <template v-if="scope.row.isNew">
            <el-button type="primary" size="small" @click="handleAdd(scope.row)">新增</el-button>
            <el-button size="small" @click="handleReset(scope.row)">重置</el-button>
          </template>
          <!-- 已有行的操作按钮 -->
          <template v-else>
            <template v-if="scope.row.isEditing">
              <el-button type="success" size="small" @click="handleSave(scope.row, scope.$index)">保存</el-button>
              <el-button size="small" @click="handleCancel(scope.row, scope.$index)">取消</el-button>
            </template>
            <template v-else>
              <el-button type="primary" size="small" @click="handleEdit(scope.row)">修改</el-button>
              <el-button type="danger" size="small" @click="handleDelete(scope.$index)">删除</el-button>
              <!-- 添加复制按钮 -->
              <el-button type="info" size="small" @click="handleCopy(scope.row)">复制</el-button>
            </template>
          </template>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
import { Message } from 'element-ui'

export default {
  name: 'EditableTable',
  props: {
    // 表格列配置
    columns: {
      type: Array,
      required: true
    },
    // 表格数据，改用 value 属性接收
    value: {
      type: Array,
      required: true
    },
    // 验证规则
    rules: {
      type: Object,
      default: () => ({})
    },
    // 字段关联关系配置
    fieldRelations: {
      type: Object,
      default: () => ({})
    },
    // 标记哪些列是共有属性
    sharedColumns: {
      type: Array,
      default: () => []
    }
  },
  
  data() {
    // 先初始化共有属性值
    const sharedValues = {}
    if (this.value && this.value.length > 0) {
      this.sharedColumns.forEach(prop => {
        sharedValues[prop] = this.value[0][prop] || ''
      })
    }

    return {
      innerTableData: [
        // 新增行，使用初始化的共有属性值
        this.createNewRow(sharedValues),
        // 初始数据
        ...this.value.map(item => ({
          ...item,
          isEditing: false,
          isNew: false,
          _original: { ...item }
        }))
      ],
      // 存储共有属性的值
      sharedValues,
      localValue: null // 添加本地数据缓存，用来保存外部数据value的最新值
    }
  },

  watch: {
    // 监听外部数据变化
    value: {
      handler(newVal) {
        // 更新本地缓存
        this.localValue = newVal

        // 保存当前所有行的编辑状态和临时数据
        const editingStates = {}
        const tempData = {}
        this.innerTableData.forEach((row, index) => {
          if (index > 0) { // 跳过新增行
            const dataIndex = index - 1
            editingStates[dataIndex] = row.isEditing
            if (row.isEditing) {
              // 保存正在编辑的行的临时数据
              tempData[dataIndex] = { ...row }
            }
          }
        })

        // 更新内部数据，但保持新增行和编辑状态及临时数据
        const newRow = this.innerTableData[0]
        this.innerTableData = [
          newRow,
          ...newVal.map((item, index) => {
            if (editingStates[index]) {
              // 如果行正在编辑，使用保存的临时数据
              return {
                ...tempData[index],
                isEditing: true,
                isNew: false,
                
              }
            } else {
              // 非编辑状态的行使用新数据
              return {
                ...item,
                isEditing: false,
                isNew: false,
                _original: { ...item }
              }
            }
          })
        ]
      },
      deep: true
    }
  },

  methods: {
    // 修改创建新行的方法，接收初始的共有属性值
    createNewRow(initialSharedValues = null) {
      const newRow = {
        isNew: true,
        isEditing: false
      }
      this.columns.forEach(col => {
        // 如果是共有属性，使用传入的值或已存储的共有值
        if (this.sharedColumns.includes(col.prop)) {
          newRow[col.prop] = initialSharedValues ? 
            initialSharedValues[col.prop] : 
            this.sharedValues[col.prop] || ''
        } else {
          newRow[col.prop] = ''
        }
      })
      return newRow
    },

    // 验证数据
    validateRow(row) {
      for (const key in this.rules) {
        const rules = this.rules[key]
        const value = row[key]
        
        for (const rule of rules) {
          if (rule.required && !value) {
            Message.error(rule.message || '该字段不能为空')
            return false
          }
          if (rule.validator && !rule.validator(value)) {
            Message.error(rule.message || '验证失败')
            return false
          }
        }
      }
      return true
    },

    // 判断字段是否禁用
    isFieldDisabled(row, prop) {
      const column = this.columns.find(col => col.prop === prop)
      
      // 如果字段默认禁用，且存在关联关系配置
      if (column.defaultDisabled && this.fieldRelations[prop]) {
        const relation = this.fieldRelations[prop]
        const dependValue = row[relation.dependOn]
        
        // 根据关联字段的值判断是否启用
        return !relation.enableWhen(dependValue)
      }
      
      // 如果正在编辑或是新行，且字段没有默认禁用，则可编辑
      return !(row.isEditing || row.isNew)
    },

    // 处理选择框变化
    handleSelectChange(row, prop) {
      // 如果是共有属性，且不是新增行，才更新所有行的值
    //   if (this.sharedColumns.includes(prop) && !row.isNew) {
    //     const newValue = row[prop]
    //     this.updateSharedValue(prop, newValue)
    //   }
      
      // 触发事件
      this.$emit('select-change', { row, prop, value: row[prop] })
    },

    // 修改更新共有属性值的方法
    updateSharedValue(prop, value) {
        debugger;
      // 更新存储的共有值
      this.$set(this.sharedValues, prop, value)

      // 创建新的内部数据数组，确保响应式更新
      const newInnerTableData = []
      this.innerTableData.forEach(row => {
        if (row.isNew) {
          // 保持新增行不变
          newInnerTableData.push(row)
        } else {
          // 更新其他行的共有属性
          const updatedRow = {
            ...row,
            [prop]: value
          }
          newInnerTableData.push(updatedRow)
        }
      })
      // 使用新数组替换原数组，触发响应式更新
      this.innerTableData = newInnerTableData

      // 更新父组件数据
      const newData = this.value.map(item => ({
        ...item,
        [prop]: value
      }))

      // 先更新本地数据，再触发事件
      this.localValue = newData // 添加本地缓存
      this.$emit('input', newData)
    },

    // 修改新增行方法
    handleAdd(row) {
      if (!this.validateRow(row)) return
      debugger;
      // 先检查新增行中是否修改了共有属性，并更新
      this.sharedColumns.forEach(prop => {
        if (row[prop] !== this.sharedValues[prop]) {
          // 更新共有属性值
          this.updateSharedValue(prop, row[prop])
        }
      })

      // 创建新数据对象，使用最新的共有属性值
      const newData = { ...row }
      delete newData.isNew
      delete newData.isEditing

      // 发送更新事件，使用新数组触发响应式更新
      const updatedValue = [...(this.localValue || this.value), newData]
      this.localValue = updatedValue
      this.$emit('input', updatedValue)
      
      // 重置新增行，使用最新的共有属性值
      const newRow = this.createNewRow()
      Object.assign(row, newRow)
      
      this.$emit('add', newData)
    },

    // 修改重置方法
    handleReset(row) {
      Object.assign(row, this.createNewRow())
    },

    // 编辑
    handleEdit(row) {
      row.isEditing = true
      row._original = { ...row }
      // 强制表格重新渲染，以更新合并状态
      this.$nextTick(() => {
        this.$forceUpdate()
      })
    },

    // 保存
    handleSave(row, rowIndex) {
      if (!this.validateRow(row)) return
      debugger;
      // 检查是否有共有属性被修改
      this.sharedColumns.forEach(prop => {
        if (row[prop] !== this.sharedValues[prop]) {
          this.updateSharedValue(prop, row[prop])
        }
      })

      row.isEditing = false
      this.innerTableData[rowIndex].isEditing = false;
      delete row._original

      // 使用本地缓存的最新数据
      const newData = [...(this.localValue || this.value)]
      newData[rowIndex-1] = { ...row }
      
      // 更新本地缓存并触发事件
      this.localValue = newData
      this.$emit('input', newData)
      this.$emit('save', row)

      // 强制表格重新渲染，以更新合并状态
      this.$nextTick(() => {
        this.$forceUpdate()
      })
    },

    // 取消
    handleCancel(row, rowIndex) {
      debugger;
      Object.assign(row, row._original)
      this.innerTableData[rowIndex].isEditing = false;
      row.isEditing = false
      this.sharedColumns.forEach(prop => {
        if (row[prop] !== this.sharedValues[prop]) {
          this.updateSharedValue(prop, row[prop])
        }
      })

      // 强制表格重新渲染，以更新合并状态
      this.$nextTick(() => {
        this.$forceUpdate()
      })
    },

    // 删除
    handleDelete(index) {
      // 因为表格的第一行是新增行，所以实际数据的索引需要减1
      const dataIndex = index - 1
      const newData = [...this.value]
      newData.splice(dataIndex, 1)
      this.localValue = newData
      this.$emit('input', newData)
      this.$emit('delete', dataIndex)
    },

    // 处理按钮点击
    async handleButtonClick(row, col) {
      try {
        // 如果列配置中提供了 api 方法，则调用它
        if (col.api) {
          const result = await col.api(row)
          // 更新字段值
          row[col.prop] = result
          // 触发变化事件
          this.$emit('button-click', { row, prop: col.prop, value: result })
        }
      } catch (error) {
        console.error('API调用失败:', error)
        Message.error('获取数据失败')
      }
    },

    // 修改处理单元格合并的方法
    handleSpanMethod({ row, column, rowIndex }) {
      // 如果是共有属性列且不是新增行
      if (this.sharedColumns.includes(column.property) && !row.isNew) {
        // 检查是否有行处于编辑状态
        const hasEditingRow = this.innerTableData.some(row => row.isEditing)
        
        // 如果没有行在编辑状态，才进行合并
        if (!hasEditingRow) {
          if (rowIndex === 1) { // 第一个数据行
            return {
              rowspan: this.innerTableData.length - 1, // 合并所有数据行
              colspan: 1
            }
          } else {
            return {
              rowspan: 0,
              colspan: 0
            }
          }
        }
      }
    },

    // 添加处理复制的方法
    handleCopy(row) {
      // 深拷贝当前行数据
      const copiedData = JSON.parse(JSON.stringify(row))
      
      // 移除编辑状态相关的属性
      delete copiedData.isEditing
      delete copiedData.isNew
      delete copiedData._original

      // 使用本地缓存的最新数据
      const newData = [...(this.localValue || this.value), copiedData]
      
      // 更新本地缓存
      this.localValue = newData
      
      // 更新父组件数据
      this.$emit('input', newData)
      
      // 触发复制事件
      this.$emit('copy', copiedData)
      
      // 更新内部数据
      this.innerTableData = [
        this.innerTableData[0], // 保持新增行
        ...newData.map(item => ({
          ...item,
          isEditing: false,
          isNew: false,
          _original: { ...item }
        }))
      ]
    }
  }
}
</script>

<style scoped>
.editable-table {
  margin: 20px;
}

.input-button-wrapper {
  display: flex;
  align-items: center;
  gap: 8px;
}

/* 添加共有属性列的样式 */
.shared-column {
  background-color: #f5f7fa;
}

/* 调整操作列按钮间距 */
.el-button + .el-button {
  margin-left: 6px;
}
</style> 