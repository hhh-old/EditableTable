<template>
  <div>
    <editable-table
      v-model="tableData"
      :columns="columns"
      :rules="rules"
      :field-relations="fieldRelations"
      :shared-columns="['sharedProp']"
      @add="handleAdd"
      @save="handleSave"
      @delete="handleDelete"
      @select-change="handleSelectChange"
      @button-click="handleButtonClick"
    />
    
    <!-- 可以显示或使用表格数据 -->
    <div class="data-preview">
      <h3>当前表格数据：</h3>
      <pre>{{ JSON.stringify(tableData, null, 2) }}</pre>
    </div>
  </div>
</template>

<script>
import EditableTable from './components/EditableTable.vue'

export default {
  components: {
    EditableTable
  },
  
  data() {
    return {
      // 表格数据移到 data 中管理
      tableData: [
        { name: '张三', type: 'A', value: '', extraInfo: '',autoValue: '', sharedProp: '共有值' },
        { name: '李四', type: 'B', value: '200', extraInfo: '',autoValue: '', sharedProp: '共有值' }
      ],
      
      columns: [
        {
          prop: 'name',
          label: '姓名',
          width: '120'
        },
        {
          prop: 'type',
          label: '类型',
          type: 'select',
          options: [
            { label: '类型A', value: 'A' },
            { label: '类型B', value: 'B' }
          ]
        },
        {
          prop: 'value',
          label: '值'
        },
        {
          prop: 'extraInfo',
          label: '附加信息',
          // 这个字段默认禁用，只有在特定条件下才能编辑
          defaultDisabled: true
        },
        {
          prop: 'autoValue',
          label: '自动获取值',
          type: 'input-button',
          buttonText: '获取数据',
          api: async (row) => {
            return new Promise((resolve) => {
              setTimeout(() => {
                resolve(`自动生成的值-${Date.now()}`)
              }, 1000)
            })
          }
        },
        {
          prop: 'sharedProp',
          label: '共有属性',
          width: '120'
        }
      ],

      rules: {
        name: [
          { required: true, message: '请输入姓名' }
        ],
        type: [
          { required: true, message: '请选择类型' }
        ]
      },

      // 定义字段之间的关联关系
      fieldRelations: {
        // extraInfo 字段的可编辑状态依赖于 type 字段的值
        extraInfo: {
          dependOn: 'type',
          // 当 type 的值为 'B' 时才可编辑
          enableWhen: value => value === 'B'
        }
      }
    }
  },

  methods: {
    handleAdd(row) {
      console.log('新增:', row)
      // 可以在这里处理新增后的逻辑
      // tableData 会自动更新，因为使用了 v-model
    },

    handleSave(row) {
      console.log('保存:', row)
      // 可以在这里处理保存后的逻辑
    },

    handleDelete(index) {
      console.log('删除索引:', index)
      // 可以在这里处理删除后的逻辑
    },

    handleSelectChange({ row, prop, value }) {
      if (prop === 'type') {
        if (value === 'A') {
          row.value = '100'
          row.extraInfo = ''
        }
      } else if(prop === 'name'){
        if(value === '张三'){
          row.value = '200'
          row.extraInfo = '张三的附加信息'
        }
      }
    },

    handleButtonClick({ row, prop, value }) {
      console.log('按钮点击后的值:', { row, prop, value })
    }
  }
}
</script>

<style>
.data-preview {
  margin: 20px;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.data-preview pre {
  background-color: #f5f5f5;
  padding: 10px;
  border-radius: 4px;
}
</style>
