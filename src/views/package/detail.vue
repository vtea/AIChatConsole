<template>
  <el-drawer
    :title="title"
    size="60%"
    :visible="show"
    :modal-append-to-body="true"
    :before-close="handleClose"
    style="height: 100%"
  >
    <div style="padding: 20px; overflow-y: auto;">
      <el-form ref="form" label-width="220px" style="padding-right: 180px">
        <el-form-item v-if="packageEntity.uuid" label="uuid">
          <el-input v-model="packageEntity.uuid" disabled />
        </el-form-item>
        <el-form-item label="标题">
          <el-input v-model="packageEntity.title" type="textarea" autosize :disabled="disabled" />
        </el-form-item>
        <el-form-item label="副标题">
          <el-input v-model="packageEntity.subTitle" type="textarea" autosize :placeholder="defaultSubTitle" :disabled="disabled" />
        </el-form-item>
        <el-form-item label="类型">
          <el-select v-model="packageEntity.calcTypeId" :disabled="disabled">
            <el-option label="总额" :value="1" />
            <el-option label="每天" :value="2" />
            <el-option label="每小时" :value="3" />
            <el-option label="每3小时" :value="4" />
          </el-select>
        </el-form-item>
        <el-form-item label="置顶值">
          <span style="color: #aaa;">{{ packageEntity.top }}（{{ getDateTimeText(packageEntity.top) }}）</span>
        </el-form-item>
        <el-form-item label="状态">
          <span style="margin-right: 10px">
            <el-tag v-if="packageEntity.state == 10" type="success">上架</el-tag>
            <el-tag v-else-if="!packageEntity.id || packageEntity.state == 0" type="primary">草稿</el-tag>
            <el-tag v-else-if="packageEntity.state == 20" type="warning">下架</el-tag>
            <el-tag v-else-if="packageEntity.state == 30" type="info">已删除</el-tag>
          </span>
          <el-button v-if="packageEntity.id && [0, 10, 20].includes(packageEntity.state)" type="primary" @click.stop="handleToggleEnable()">
            {{ packageEntity.state == 10 ? '下架' : '上架' }}
          </el-button>
        </el-form-item>
        <el-form-item label="tokens">
          <el-input-number v-model="packageEntity.tokens" :disabled="disabled" />
        </el-form-item>
        <el-form-item label="普通聊天积分">
          <el-input-number v-model="packageEntity.chatCount" :disabled="disabled" />
        </el-form-item>
        <el-form-item label="高级聊天积分">
          <el-input-number v-model="packageEntity.advancedChatCount" :disabled="disabled" />
        </el-form-item>
        <el-form-item label="绘图积分">
          <el-input-number v-model="packageEntity.drawCount" :disabled="disabled" />
        </el-form-item>
        <el-form-item label="天数">
          <el-input-number v-model="packageEntity.days" :disabled="disabled" />
        </el-form-item>
        <el-form-item label="价格（元）">
          <el-input v-model="packageEntity.price" :disabled="disabled" />
        </el-form-item>
        <el-form-item>
          <el-alert v-if="packageEntity.state === 10" title="请先下架" type="info" :closable="false" />
          <el-button type="primary" :disabled="disabled" @click="handleSubmit">
            {{ loading ? '操作中……' : packageEntity.id ? '保存' : '创建' }}
          </el-button>
        </el-form-item>
      </el-form>
      <el-form>
        <div
          style="margin-bottom: 100px;"
        >
          <el-alert
            title="以上所有积分（及tokens）均表示每天的积分。每天剩余使用积分将被清零，不结转至下一个自然日。"
            type="info"
            style="margin-bottom: 5px;"
            :closable="false"
          />
          <el-alert
            title="天数是指，用户从购买之日起算，截止设定的天数前可用，过期自动失效。建议月卡设定天数为31天，年卡设定天数为366天。"
            type="info"
            :closable="false"
          />
        </div>
        <el-form-item v-if="packageEntity.id">
          <el-button type="danger" @click="handleDelete">
            删除
          </el-button>
        </el-form-item>
      </el-form>

    </div>
  </el-drawer>
</template>

<script>

// import AiTable from '@/components/Table'
// import { getBalanceRecordByUserId, increaseBalance } from '@/api/balance'
// import ChangePassword from '../user/password'
import { createPackage, updatePackage, putOnSales, putOffShelves, deletePackage } from '@/api/package'

export default {
  name: 'PackageDetail',
  components: { },
  props: {
    show: {
      type: Boolean,
      default: false
    },
    packageEntity: {
      type: Object,
      default: () => []
    }
  },
  data() {
    return {
      loading: false,
      drawer: true
    }
  },
  computed: {
    title() {
      return this.packageEntity.title + '详情'
    },
    defaultSubTitle() {
      if (!this.packageEntity.calcTypeId) {
        return ''
      }
      const text = []
      const prefix = ({
        1: '',
        2: '每天',
        3: '每小时',
        4: '每3小时'
      })[this.packageEntity.calcTypeId]
      if (this.packageEntity.tokens) {
        text.push(`${prefix} ${this.packageEntity.tokens} token`)
      }
      if (this.packageEntity.chatCount) {
        text.push(`${prefix} ${this.packageEntity.chatCount} 次普通聊天（GPT3.5）`)
      }
      if (this.packageEntity.advancedChatCount) {
        text.push(`${prefix} ${this.packageEntity.advancedChatCount} 次高级聊天（GPT4）`)
      }
      if (this.packageEntity.drawCount) {
        text.push(`${prefix} ${this.packageEntity.drawCount} 次AI绘画`)
      }
      if (this.packageEntity.days) {
        text.push(`有效期：${this.packageEntity.days}天`)
      }
      return text.join('，')
    },
    disabled() {
      return (this.loading) || this.packageEntity.state === 10
    }
  },
  mounted() {
    // this.reload()
  },
  methods: {
    reload() { // 对外接口
      if (!this.packageEntity.id) {
        return
      }
    },
    handleClose() {
      this.$emit('close')
    },
    handleRefresh() {
      this.reload()
    },
    handleSubmit() {
      if (!this.packageEntity.title) {
        this.$message.error('标题不能为空！')
        return
      }
      if (!this.packageEntity.calcTypeId) {
        this.$message.error('请选择类型')
        return
      }
      if (this.packageEntity.days <= 0) {
        this.$message.error('天数必须是正数！')
        return
      }
      if (!this.packageEntity.price) {
        this.$message.error('价格不能为空！')
        return
      }
      if (this.packageEntity.id) {
        if (this.packageEntity.state === 10) {
          this.$message.error('请先下架！')
          return
        }
        this.loading = true
        updatePackage(this.packageEntity).then((resp) => {
          this.$message.success('操作成功！')
          this.$emit('changed', this.packageEntity.id)
        }).finally(() => {
          this.loading = false
        })
      } else {
        this.loading = true
        createPackage({
          title: this.packageEntity.title,
          subTitle: this.packageEntity.subTitle,
          tokens: this.packageEntity.tokens,
          chatCount: this.packageEntity.chatCount,
          advancedChatCount: this.packageEntity.advancedChatCount,
          drawCount: this.packageEntity.drawCount,
          days: this.packageEntity.days,
          price: this.packageEntity.price,
          calcTypeId: this.packageEntity.calcTypeId
        }).then((resp) => {
          this.$message.success('创建成功！')
          this.$emit('changed', resp.data.id)
        }).finally(() => {
          this.loading = false
        })
      }
    },
    handleToggleEnable() {
      // console.log('handleToggleEnable')
      const row = this.packageEntity
      if (row.state === 0 || row.state === 20) {
        // 说明要上架
        if (!row.title) {
          this.$message.error('请输入标题！')
          return
        }
        if (!row.calcTypeId) {
          this.$message.error('请选择类型')
          return
        }
        if (!row.price) {
          this.$message.error('请输入价格')
          return
        }
      }
      this.$confirm(
        row.state === 10 ? ('下架后用户将无法购买此套餐，确定下架' + row.title + '吗？') : ('上架后套餐信息无法修改，确定上架' + row.title + '吗？'),
        '操作确认',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }
      ).then(async() => {
        if (row.state === 10) {
          putOffShelves(row.uuid).then(() => {
            this.$message.success('操作成功！')
            this.$emit('changed', row.id)
          })
        } else {
          putOnSales(row.uuid).then(() => {
            this.$message.success('操作成功！')
            this.$emit('changed', row.id)
          })
        }
      })
    },
    getDateTimeText(top) {
      const total = new Date(top).toISOString()
      return total.slice(0, 10) + ' ' + total.slice(11, 19)
    },
    handleDelete() {
      const row = this.packageEntity
      this.$confirm('确定删除' + row.title + '吗？',
        '删除确认',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }
      ).then(async() => {
        deletePackage(row.uuid).then(() => {
          this.$message.success('操作成功！')
          this.$emit('changed')
          this.$emit('close')
        })
      })
    }
  }
}
</script>

<style lang="scss" scoped>
::v-deep .el-drawer__body {
  height: calc(100% - 77px);
  overflow-y: auto;
}
</style>
