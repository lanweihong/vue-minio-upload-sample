<template>
  <div class="container">
    <h2>Minio 上传示例</h2>
    <el-upload
      class="upload-demo"
      ref="upload"
      action="https://jsonplaceholder.typicode.com/posts/"
      :on-remove="handleRemove"
      :on-change="handleFileChange"
      :file-list="uploadFileList"
      :show-file-list="false"
      :auto-upload="false">
      <el-button slot="trigger" type="primary" plain>选择文件</el-button>
      <el-button style="margin-left: 10px;" type="success" @click="handleUpload" plain>上传</el-button>
      <el-button type="danger" @click="clearFileHandler" plain>清空</el-button>
    </el-upload>
    <!-- 文件列表 -->
    <div class="file-list-wrapper">
      <el-collapse>
        <el-collapse-item v-for="(item, index) in uploadFileList" :key="index">
          <template slot="title">
            <div class="upload-file-item">
              <div class="file-info-item file-name">文件名：{{ item.name }}</div>
              <div class="file-info-item file-size">文件大小：{{ item.size | transformByte }}</div>
              <div class="file-info-item file-progress">
                <span class="file-progress-label">文件进度：</span>
                <el-progress :percentage="item.uploadProgress" class="file-progress-value" />
              </div>
              <div class="file-info-item file-size"><span>状态：</span>
                <el-tag v-if="item.status === '等待上传'" size="medium" type="info">等待上传</el-tag>
                <el-tag v-else-if="item.status === '校验MD5'" size="medium" type="warning">校验MD5</el-tag>
                <el-tag v-else-if="item.status === '正在上传'" size="medium">正在上传</el-tag>
                <el-tag v-else-if="item.status === '上传成功'" size="medium" type="success">上传完成</el-tag>
                <el-tag v-else size="medium" type="danger">上传错误</el-tag>
              </div>
          </div>
          </template>
          <div class="file-chunk-list-wrapper">
            <!-- 分片列表 -->
            <el-table
              :data="item.chunkList"
              max-height="400"
              style="width: 100%">
              <el-table-column
              prop="chunkNumber"
              label="分片序号"
              width="180">
              </el-table-column>
              <el-table-column
              prop="progress"
              label="上传进度">
              <template v-slot="{ row }">
                <el-progress v-if="!row.status || row.progressStatus === 'normal'" :percentage="row.progress" />
                <el-progress v-else :percentage="row.progress" :status="row.progressStatus" :text-inside="true" :stroke-width="16" />
              </template>
              </el-table-column>
              <el-table-column
                prop="status"
                label="状态"
                width="180">
              </el-table-column>
            </el-table>
          </div>
         
        </el-collapse-item>
      </el-collapse>
    </div>
  </div>
</template>

<script>
import SparkMD5 from 'spark-md5'
import axios from 'axios'
const FILE_UPLOAD_ID_KEY = 'file_upload_id'
const chunkSize = 10 * 1024 * 1024
let currentFileIndex = 0
const FileStatus = {
  wait: '等待上传',
  getMd5: '校验MD5',
  uploading: '正在上传',
  success: '上传成功',
  error: '上传错误'
}
  export default {
    data () {
      return {
        changeDisabled: false,
        uploadDisabled: false,
        // 上传并发数
        simultaneousUploads: 3,
        uploadIdInfo: null,
        uploadFileList: [],
        retryList: []
      }
    },
    methods: {
      handleUpload() {
        const self = this
        const files = this.uploadFileList
        if (files.length === 0) {
          this.$message.error('请先选择文件')
          return
        }
        // 当前操作文件
        const currentFile = files[currentFileIndex]
        currentFile.status = FileStatus.getMd5
        // 1. 计算MD5
        this.getFileMd5(currentFile.raw, async (md5) => {
          // 2. 检查是否已上传
          const checkResult = await self.checkFileUploadedByMd5(md5)
          // 已上传
          if (checkResult.data.status === 1) {
            self.$message.success(`上传成功，文件地址：${checkResult.data.url}`)
            console.log('文件访问地址：' + checkResult.data.url)
            currentFile.status = FileStatus.success
            currentFile.uploadProgress = 100
            return
          } else if (checkResult.data.status === 2) {  // "上传中" 状态
            // 获取已上传分片列表
            let chunkUploadedList = checkResult.data.chunkUploadedList
            currentFile.chunkUploadedList = chunkUploadedList  
          } else {   // 未上传
            console.log('未上传')
          }

          console.log('文件MD5：' + md5)
          // 3. 正在创建分片
          let fileChunks = self.createFileChunk(currentFile.raw, chunkSize)
          
          let param = {
            fileName: currentFile.name,
            fileSize: currentFile.size,
            chunkSize: chunkSize,
            fileMd5: md5,
            contentType: 'application/octet-stream'
          }
          // 4. 获取上传url
          let uploadIdInfoResult = await self.getFileUploadUrls(param)
          self.uploadIdInfo = uploadIdInfoResult.data.data
          self.saveFileUploadId(uploadIdInfoResult.data.data)
          let uploadUrls = uploadIdInfoResult.data.data.uploadUrls
          if (fileChunks.length !== uploadUrls.length) {
            self.$message.error('文件分片上传地址获取错误')
            return
          }
          self.$set(currentFile, 'chunkList', [])
          fileChunks.map((chunkItem, index) => {
            currentFile.chunkList.push({
              chunkNumber: index + 1,
              chunk: chunkItem,
              uploadUrl: uploadUrls[index],
              progress: 0,
              status: '—'
            })
          })
          let tempFileChunks = []
          currentFile.chunkList.forEach((item) => {
            tempFileChunks.push(item)
          })
          currentFile.status = FileStatus.uploading
          // 处理分片列表，删除已上传的分片
          tempFileChunks = self.processUploadChunkList(tempFileChunks)
          // 5. 上传
          await self.uploadChunkBase(tempFileChunks)
          console.log('上传完成')
          // 6. 合并文件
          const mergeResult = await self.mergeFile({
            uploadId: self.uploadIdInfo.uploadId,
            fileName: currentFile.name,
            md5: md5
          })
          if (!mergeResult.success) {
            currentFile.status = FileStatus.error
            self.$message.error(mergeResult.error)
          } else {
            currentFile.status = FileStatus.success
            console.log('文件访问地址：' + mergeResult.data.url)
            self.$message.success(`上传成功，文件地址：${mergeResult.data.url}`)
          }
        })   
      },
      clearFileHandler() {
        this.uploadFileList = []
        this.uploadIdInfo = null
      },
      handleFileChange(file, fileList) {
        this.uploadFileList = fileList
        this.uploadFileList.forEach((item) => {
          // 初始化自定义属性
          this.initFileProperties(item)
        })
      },
      initFileProperties(file) {
        file.chunkList = []
        file.status = FileStatus.wait
        file.progressStatus = 'warning'
        file.uploadProgress = 0
      },
      handleRemove(file, fileList) {
        this.uploadFileList = fileList
      },
      /**
       * 分片读取文件 MD5
       */
      getFileMd5(file, callback) {
        const blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
        const fileReader = new FileReader()
        // 计算分片数
        const totalChunks = Math.ceil(file.size / chunkSize)
        console.log('总分片数：' + totalChunks)
        let currentChunk = 0
        const spark = new SparkMD5.ArrayBuffer()
        loadNext()
        fileReader.onload = function (e) {
          try {
            spark.append(e.target.result)
          } catch (error) {
            console.log('获取Md5错误：' + currentChunk)
          }
          if (currentChunk < totalChunks) {
            currentChunk++
            loadNext()
          } else {
            callback(spark.end())
          }
        }
        fileReader.onerror = function () {
          console.warn('读取Md5失败，文件读取错误')
        }
        function loadNext () {
          const start = currentChunk * chunkSize
          const end = ((start + chunkSize) >= file.size) ? file.size : start + chunkSize
          // 注意这里的 fileRaw
          fileReader.readAsArrayBuffer(blobSlice.call(file, start, end))
        }
      },
      /**
       * 文件分片
       */
      createFileChunk(file, size = chunkSize) {
        const fileChunkList = []
        let count = 0
        while(count < file.size) {
          fileChunkList.push({
            file: file.slice(count, count + size),
          })
          count += size
        }
        return fileChunkList
      },
      /**
       * 处理即将上传的分片列表，判断是否有已上传的分片，有则从列表中删除
       */
      processUploadChunkList(chunkList) {
        const currentFile = this.uploadFileList[currentFileIndex]
        let chunkUploadedList = currentFile.chunkUploadedList
        if (chunkUploadedList === undefined || chunkUploadedList === null || chunkUploadedList.length === 0) {
          return chunkList
        }
        // 
        for (let i = chunkList.length - 1; i >= 0; i--) {
          const chunkItem = chunkList[i]
          for (let j = 0; j < chunkUploadedList.length; j++) {
            if (chunkItem.chunkNumber === chunkUploadedList[j]) {
              chunkList.splice(i, 1)
              break
            }
          }
        }
        return chunkList
      },
      uploadChunkBase(chunkList) {
        const self = this
        let successCount = 0
        let totalChunks = chunkList.length
        return new Promise((resolve, reject) => {
          const handler = () => {
            if (chunkList.length) {
              const chunkItem = chunkList.shift()
              // 直接上传二进制，不需要构造 FormData，否则上传后文件损坏
              axios.put(chunkItem.uploadUrl, chunkItem.chunk.file, {
                // 上传进度处理
                onUploadProgress: self.checkChunkUploadProgress(chunkItem),
                headers: {
                  'Content-Type': 'application/octet-stream'
                }
              }).then(response => {
                if (response.status === 200) {
                  console.log('分片：' + chunkItem.chunkNumber + ' 上传成功')
                  successCount++
                  // 继续上传下一个分片
                  handler()
                } else {
                  console.log('上传失败：' + response.status + '，' + response.statusText)
                }
              }).catch(error => {
                // 更新状态
                console.log('分片：' + chunkItem.chunkNumber + ' 上传失败，' + error)
                // 重新添加到队列中
                chunkList.push(chunkItem)
                handler()
              })
            }
            if (successCount >= totalChunks) {
              resolve()
            }
          }
          // 并发
          for (let i = 0; i < this.simultaneousUploads; i++) {
            handler()
          }
        })
      },
      getFileUploadUrls(fileParam) {
        let url = `http://127.0.0.1:8027/upload?_=${Math.random()}`
        return axios.post(url, fileParam)
      },
      saveFileUploadId(data) {
        localStorage.setItem(FILE_UPLOAD_ID_KEY, data)
      },
      checkFileUploadedByMd5(md5) {
        let url = `http://127.0.0.1:8027/upload/check?md5=${md5}`
        return new Promise((resolve, reject) => {
          axios.get(url).then((response) => {
            resolve(response.data)
          }).catch(error => {
            reject(error)
          })
        })
      },
      /**
       * 合并文件
       */
      mergeFile(file) {
        const self = this
        let url = `http://127.0.0.1:8027/upload/merge?uploadId=${file.uploadId}&fileName=${file.fileName}&md5=${file.md5}`
        return new Promise((resolve, reject) => {
          axios.post(url).then(response => {
            let data = response.data
            if (!data.success) {
              file.status = FileStatus.error
              resolve(data)
            } else {
              file.status = FileStatus.success
              resolve(data)
            }
          }).catch(error => {
            self.$message.error('合并文件失败：' + error)
            file.status = FileStatus.error
            reject()
          })
        })
      },
      /**
       * 检查分片上传进度
       */
      checkChunkUploadProgress(item) {
        return p => {
          item.progress = parseInt(String((p.loaded / p.total) * 100))
          this.updateChunkUploadStatus(item)
        }
      },
      updateChunkUploadStatus(item) {
        let status = FileStatus.uploading
        let progressStatus = 'normal'
        if (item.progress >= 100) {
          status = FileStatus.success
          progressStatus = 'success'
        }
        let chunkIndex = item.chunkNumber - 1
        let currentChunk = this.uploadFileList[currentFileIndex].chunkList[chunkIndex]
        // 修改状态
        currentChunk.status = status
        currentChunk.progressStatus = progressStatus
        // 更新状态
        this.$set(this.uploadFileList[currentFileIndex].chunkList, chunkIndex, currentChunk)
        // 获取文件上传进度
        this.getCurrentFileProgress()
      },
      getCurrentFileProgress() {
        const currentFile = this.uploadFileList[currentFileIndex]
        if (!currentFile || !currentFile.chunkList) {
          return
        }
        const chunkList = currentFile.chunkList
        const uploadedSize = chunkList.map((item) => item.chunk.file.size * item.progress).reduce((acc, cur) => acc + cur)
        // 计算方式：已上传大小 / 文件总大小
        let progress = parseInt((uploadedSize / currentFile.size).toFixed(2))
        currentFile.uploadProgress = progress
        this.$set(this.uploadFileList, currentFileIndex, currentFile)
      }
    },
    filters: {
      transformByte(size) {
        if (!size) {
          return '0B'
        }
        const unitSize = 1024
        if (size < unitSize) {
          return size + ' B'
        }
        // KB
        if (size < Math.pow(unitSize, 2)) {
          return (size / unitSize).toFixed(2) + ' K';
        }
        // MB
        if (size < Math.pow(unitSize, 3)) {
          return (size / Math.pow(unitSize, 2)).toFixed(2) + ' MB'
        }
        // GB
        if (size < Math.pow(unitSize, 4)) {
          return (size / Math.pow(unitSize, 3)).toFixed(2) + ' GB';
        }
        // TB
        return (size / Math.pow(unitSize, 4)).toFixed(2) + ' TB';
      }
    }
  }
</script>

<style>
.container {
  width: 1000px;
  margin: 0 auto;
}
.file-list-wrapper {
  margin-top: 20px;
}
h2 {
  text-align: center;
}
.file-info-item {
  margin: 0 20px;
}
.upload-file-item {
  display: flex;
}
.file-progress {
  display: flex;
  align-items: center;
}
.file-progress-value {
  width: 250px;
}
  .uploader-example {
    width: 880px;
    padding: 15px;
    margin: 40px auto 0;
    font-size: 12px;
    box-shadow: 0 0 10px rgba(0, 0, 0, .4);
  }
  .uploader-example .uploader-btn {
    margin-right: 4px;
  }
  .uploader-example .uploader-list {
    max-height: 440px;
    overflow: auto;
    overflow-x: hidden;
    overflow-y: auto;
  }
</style>