<template>
  <el-dialog
    :title="title"
    :visible.sync="dialogFormVisible"
    width="800px"
    class="import-excel"
    @close="close"
  >
    <el-upload
      ref="excelImport"
      drag
      accept=".xls,.xlsx"
      style="width: 100%"
      :action="importExcelUrl"
      :multiple="false"
      :before-upload="beforeUpload"
      :http-request="handleImport"
      :on-success="onSuccess"
      :on-error="onError"
      :on-progress="onProcess"
    >
      <i class="el-icon-upload"></i>
      <div class="el-upload__text">将文件拖到此处，或<em>点击导入</em></div>
      <div class="el-upload__tip" slot="tip">只能上传xls/xlsx文件，且不超过5MB</div>
    </el-upload>

    <div slot="footer" class="dialog-footer">
      <el-button type="primary" @click="downloadExcelTemplate">下载模版</el-button>
      <el-button @click="close">关 闭</el-button>
    </div>
  </el-dialog>
</template>

<script>
  import { doDownloadTemplate, doImportExcel } from "@/api/gentest/carinfo/TestCarManagement";
  import {isNull} from "@/utils/validate";
  import {random} from "@/utils";

  export default {
    name: "TestCarManagementImport",
    data() {
      return {
        title: "导入Excel",
        importExcelUrl: '',
        dialogFormVisible: false,
        loadProgress: 0, // 动态显示进度条
        progressFlag: false, // 关闭进度条,
        progressMap: {}
      };
    },
    created() {},
    mounted() {},
    methods: {
      show() {
        this.dialogFormVisible = true;
      },
      close() {
        this.dialogFormVisible = false;
        this.$refs["excelImport"].clearFiles();
      },
      // 下载模版
      downloadExcelTemplate() {
        doDownloadTemplate();
      },
      // 上传成功
      onSuccess(response, file, fileList){
        this.successProcess(file.uid);
        this.$emit("fetchData");
      },
      // 上传失败
      onError(err, file, fileList){
        this.errorProcess(file.uid);
      },
      // 进度条
      onProcess(event, file, fileList) {
        file.status = 'uploading';
        file.percentage = 0;
        this.progressMap[file.uid] = {
          file: file,
        }
        this.autoLoadingProcess(file.uid);
      },

      // 导入文件限制验证
      beforeUpload(file) {
        let testMsg = file.name.substring(file.name.lastIndexOf('.')+1)
        const extension = testMsg === 'xls'
        const extension2 = testMsg === 'xlsx'
        const isLt2M = file.size / 1024 / 1024 < 5
        if(!extension && !extension2) {
          this.$baseMessage('上传文件只能是 xls、xlsx格式!', "warning");
        }
        if(!isLt2M) {
          this.$baseMessage('上传文件大小不能超过 5MB!', "warning");
        }
        return (extension || extension2) && isLt2M
      },
      // 自定义导入
      handleImport(params){
        if(!isNull(params)){
          let blobObject = new Blob([params.file]);
          let formData = new window.FormData()
          formData.append("file", blobObject);
          const ret = doImportExcel(formData);
          ret.then((v) => {
            const {success,msg,data} = v;
            if(success){
              this.$baseMessage(msg, "success");
              // 成功
              params.onSuccess();
            }else{
              // 文件进度 100%
              this.errorProcess(params.file.uid);
              // 失败
              params.onError();
            }
          }).catch( (e) =>{
            // 失败
            params.onError();
          });
          // 上传进度
          params.onProgress();
        }else{
          params.onError();
        }
      },

      // ==============

      successProcess(fileUid) {
        let tmp = this.progressMap[fileUid];
        if(tmp !== null && tmp !== undefined){
          try {
            window.clearTimeout(tmp.timer);
          }catch (e){}
          tmp.file.status = 'success';
          tmp.file.percentage = 100;
          delete this.progressMap[fileUid];
        }
      },
      errorProcess(fileUid) {
        let tmp = this.progressMap[fileUid];
        if(tmp !== null && tmp !== undefined){
          try {
            window.clearTimeout(tmp.timer);
          }catch (e){}
          tmp.file.status = 'fail';
          delete this.progressMap[fileUid];
        }
      },
      autoLoadingProcess(fileUid) {
        const that = this;
        let tmp = this.progressMap[fileUid];
        if(tmp !== null && tmp !== undefined){
          if(tmp.file.percentage >= 99) {
            try {
              window.clearTimeout(tmp.timer);
            }catch (e){}
          }else {
            // 如果大于 99 则 停止
            if(tmp.file.percentage + random(1, 12) > 99){
              tmp.file.percentage = 99;
            }else{
              // 进度随机增长 1 - 12
              tmp.file.percentage += random(1, 12);
            }

            // 递归增加百分比 递归时间为 随机 1-5秒
            tmp.timer = window.setTimeout(function (){
              that.autoLoadingProcess(fileUid);
            }, random(1000, 5000));
          }
        }
      },

    },
  };
</script>
