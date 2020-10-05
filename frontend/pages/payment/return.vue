<template>
  <div class="detail-container">
    <el-card>
      <P>Payment Success</P>
      <el-button
        type="primary"
        class="btn-save"
        @click="closeWindow()"
      >
        <span>Return</span>
        <i class="el-icon-right"></i>
      </el-button>
    </el-card>
  </div>
</template>
<script>
export default {
  layout: 'detail',
  methods: {
    closeWindow () {
      self.close()
      self.opener.document.getElementById('btnPayReturn').click()
    }
  }
}
</script>
<style lang="scss" scoped>
.el-card {
  text-align: center;
}
</style>
