<template>
  <el-dialog :append-to-body="true" :close-on-click-modal="false" :before-close="cancel" :visible.sync="dialog"  width="600px">
    <el-form ref="form" :model="form" :rules="rules" size="small" label-width="80px">
      <!--   上传文件   -->
      <el-form-item  label="上传">
        <el-upload
          ref="upload"
          :limit="1"
          :before-upload="beforeUpload"
          :auto-upload="false"
          :http-request="uploadFile"
          :on-success="handleSuccess"
          :on-error="handleError"
          accept="application/vnd.ms-excel, application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
          :action="baseURL+uploadApi+'/upload'">
          <div class="eladmin-upload"><i class="el-icon-upload"/> 添加文件</div>
          <div slot="tip" class="el-upload__tip">只可上传EXECL文件</div>
        </el-upload>
      </el-form-item>
      <el-form-item>
        <div>
          <h4 v-if="fileProcess.code>=0">{{fileProcess.status}}&nbsp;&nbsp;&nbsp;&nbsp;
            总数量:{{fileProcess.count}}&nbsp;&nbsp;&nbsp;&nbsp;
            完成数量:{{fileProcess.finish}}</h4>
          <div v-if="fileProcess.count>0">
            <el-progress  :text-inside="true" :stroke-width="26" :percentage="Number((fileProcess.finish*100/fileProcess.count).toFixed(2))"></el-progress>
          </div>
          <div v-if="fileProcess.code!=1&&fileProcess.finish===fileProcess.count&&fileProcess.count>0">
            <h4>正在提交数据库数据...</h4>
          </div>
        </div>
      </el-form-item>
    </el-form>
    <div slot="footer" class="dialog-footer">
      <el-button type="text" @click="cancel">取消</el-button>
      <el-button  type="primary" @click="doSubmit">确认</el-button>
    </div>
  </el-dialog>
</template>

<script>
import { getToken } from '@/utils/auth'
import axios from 'axios'
import initData from '@/mixins/initData'
import { randomString } from '@/utils/index'
export default {
  props: {
    uploadApi: {
      type: String,
      required: true
    }
  },
  mixins: [initData],
  data() {
    return {
      fileName: '', dialog: false,
      headers: { 'Authorization': 'Bearer ' + getToken() },
      form: {
        id: '',
        name: ''
      },
      baseURL: process.env.NODE_ENV === 'production' ? process.env.BASE_API+'/' : '/',
      rules: {
      },
      fileProcess:{
        status:'正在导入',
        code:-1,
        count:0,
        finish:0
      }
    }
  },
  computed: {
  },
  methods: {
    cancel() {
      this.resetForm()
    },
    doSubmit() {
      this.openFullScreen('导入',this.fileName);
      this.$refs.upload.submit()
    },
    resetForm() {
      this.dialog = false
      this.$refs['form'].resetFields()
      this.form = {
        id: '',
        name: ''
      }
      this.fileProcess = {
          status:'',
          code:-1,
          count:0,
          finish:0
      }
    },
    beforeUpload(file) {
      let isLt2M = true;
      isLt2M = file.size / 1024 / 1024 < 100;
      if (!isLt2M) {
        this.$message.error('上传文件大小不能超过 100MB!');
      }
      isLt2M = /\.xlsx?$/.test(file.name);
      if (!isLt2M) {
        this.$message.error('文件格式不正确(.xls或.xlsx)');
      }
      this.fileName = file.name;
      return isLt2M
    },
    handleSuccess(response, param,form) {
      console.log(response)
      if(response){
         var res = response.data;
        if(res.code == 0){
            this.loadingInstance.text = "";
            this.fileProcess.status = res.status;
            this.fileProcess.code = res.code;
            if('count' in res ){
               this.fileProcess.count = res.count
            }
            if('haveInsert' in  res){
              this.fileProcess.finish = res.haveInsert;
            }
            this.postFile(param,form);
        }else if(res.code == 1){
          this.loadingInstance.close();
          this.dialog = false;
          this.resetForm();
          this.$refs.upload.clearFiles();
          this.$parent.init();
          this.$notify({
            title: '文件['+param.file.name+']导入成功',
            type: 'success',
            duration: 2500
          });
        }else{
           this.handleError(res.status);
        }
      }
    },
    // 监听上传失败
    handleError(msg) {
      this.dialog = false;
      this.loadingInstance.close();
      this.resetForm();
      this.$refs.upload.clearFiles();
      this.$notify({
        title: msg,
        type: 'error',
        duration: 2500
      });
      this.fileName = ''
    },
    uploadFile(param){
      var fileObj = param.file;
      var form = new FormData();
      var id = randomString(32);
      form.append("file", fileObj);
      form.append("id", id);
      this.postFile(param,form);
    },
    postFile(param,form){
      axios.post(
        this.uploadApi+'/upload',
        form,
        {
          headers: { 'Content-Type': 'multipart/form-data','Authorization': 'Bearer ' + getToken() },
          baseURL: this.baseURL
        },
        {
          timeout:30000000
        }).then((response)=>{
        this.handleSuccess(response,param,form);
      }).catch(e=>{
        this.handleError(e.message);
      })
    }
  }
}
</script>

<style scoped>

</style>
