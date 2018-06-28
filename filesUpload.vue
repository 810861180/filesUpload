<template>
    <div>
        <!-- 上传文件 -->
        <div class="upload">
            <input type="file" id="fileupload" style="background:white; border-radius:5px;">
            <Button type="info" @click="uploadf">上传</Button>
            <p>{{status}}</p>
            <Progress :percent="percent" status="active" v-show="percentShow" style="width:36%;"></Progress>
        </div> 
        
    </div>
</template>

<script>
export default {
    data () {
        return{
            sizes: 10 * 1024 * 1024,
            url: 'http://172.20.3.20:9002',
            percent:0,
            percentShow:false,
            fileSize:0,
            status:'',
        }
    },
    methods:{
        getMd5List(index, shardCount, file, size, name, uuid, date, filemd5) {
            let self = this
            if (index < shardCount){
                let shardSize = self.sizes
                //计算每一片的起始与结束位置
                var start = index * shardSize;
                var end = Math.min(size, start + shardSize);
                // 生成 Md5 ， 将文件切割后 ， 将 切割后的 文件 读入内存中 　
                var singleFile = file.slice(start, end);
                // 计算出 md5 值
                browserMD5File(singleFile, function (err, md5) {
                    // 验证并上传
                    self.checkUploadFile(md5, uuid, name, size, shardCount, filemd5, date, index, singleFile)
                    // 索引加一
                    index = index + 1
                    // 递归执行
                    self.getMd5List(index, shardCount, file, size, name, uuid, date, filemd5)
                });
                // 计算进度
                var progress = parseInt(((index+1) / shardCount) * 100) 
                self.percent = progress
                console.log(self.percent)
                if (self.percent === 100) {
                   self.status = '准备完毕，提交中...'
                }
            }
            
            if (index == shardCount){
                console.log("计算结束 " )
                console.log("总片数" + shardCount)
            }
        },
    // 检查该文件以前是否上传过
    isUpload(file) {
        let self = this
        function getUploadDataInfo(file, uuid, filemd5, date) {
        let shardSize = self.sizes
        let name = file.name;        //文件名
        let size = file.size;        //总大小
        self.fileSize = size
        let shardCount = Math.ceil(size / shardSize);  //总片数
        // 获取 md5 列表
        self.getMd5List(0, shardCount, file, size, name, uuid, date, filemd5)
        }
        let ipport = this.url
        //构造一个表单，FormData是HTML5新增的
        var form = new FormData();
        // 计算 整个 上传文件 的 Md5 码的 值
        self.status = '准备中...'
        
        browserMD5File(file, function (err, md5) {
            // 计算后的 md5 的 结果
            form.append("md5", md5);
            //Ajax提交
            $.ajax({
                url: ipport + "/appcloudfile/isUpload",
                type: "POST",
                data: form,
                async: true,        //异步
                processData: false,  //很重要，告诉jquery不要对form进行处理
                contentType: false,  //很重要，指定为false才能形成正确的Content-Type
                success: function (data) {
                    // 获得返回结果
                    var dataObj = data;
                    // 拿到上传文件 的 文件 ID
                    var uuid = dataObj.fileId;
                    var date = dataObj.date;
                    if (dataObj.flag == "1") {
                        //没有上传过文件,开始分片上传文件
                        getUploadDataInfo(file, uuid, md5, date)
                        self.percentShow = true
                    } else if (dataObj.flag == "2") {
                        //已经上传部分
                        alert('已经上传')
                        getUploadDataInfo(file, uuid, md5, date)
                    } else if (dataObj.flag == "3") {
                        //文件已经上传过
                        alert("文件已经上传过！！");
                        self.status = ''
                    }
                }, error: function (XMLHttpRequest, textStatus, errorThrown) {
                        alert("服务器出错!");
                }
            });
        });
    },
    // 检查分片是否上传
    checkUploadFile(singleFileMd5, uuid, name, size, shardCount, fileMd5, date, i, singleFile){
        let self = this
        var form = new FormData()
        let ipport = this.url
        //check 参数 表示 这个 请求作用 是  检测分片是否上传
        form.append("action", "check");
        form.append("uuid", uuid);
        form.append("filemd5", fileMd5);
        form.append("date", date);
        form.append("name", name);
        form.append("size", size);
        form.append("total", shardCount);  //总片数
        form.append("index", i+1);        //当前是第几片
        form.append("md5", singleFileMd5);
        $.ajax({
            url: ipport + "/appcloudfile/upload",
            type: "POST",
            data: form,
            async: true,        //异步
            processData: false,  //很重要，告诉jquery不要对form进行处理
            contentType: false,  //很重要，指定为false才能形成正确的Content-Type
            success: function (data) {
                var dataObj = data;
                var flag = dataObj.flag;
                //服务器返回该分片是否上传过 1 表示 未上传过 ， 2 表示上传过
                if (flag == "1") {
                    //未上传
                    self.uploadSingleFile(singleFile, fileMd5, name, singleFileMd5, uuid, date, size, shardCount, i)
                } else if (flag == "2") {
                    //已上传
                    console.log("该分片已上传!")
                }
            }, error: function (XMLHttpRequest, textStatus, errorThrown) {
                console.log("服务器出错, 检查是否上传出错!")
            }
        });
    },
    // 开始上传单个分片
    uploadSingleFile(singleFile, fileMd5, name, md5, uuid, date , size, shardCount, i) {
        let self = this
        let ipport = this.url
        var form = new FormData();
        //upload 参数 表示 这个 请求作用 是  直接上传分片
        form.append("action", "upload");
        form.append("data", singleFile);  //slice方法用于切出文件的一部分
        form.append("uuid", uuid);
        form.append("filemd5", fileMd5);
        form.append("date", date);
        form.append("name", name);
        form.append("size", size);
        form.append("total", shardCount);  //总片数
        form.append("index", i+1);        //当前是第几片
        // 计算出 md5 值
        form.append("md5", md5);
        //Ajax提交
        $.ajax({
            url: ipport + "/appcloudfile/upload",
            type: "POST",
            data: form,
            async: true,        //异步
            processData: false,  //很重要，告诉jquery不要对form进行处理
            contentType: false,  //很重要，指定为false才能形成正确的Content-Type
            success: function (data) {
                data.uploadStatus
                // 计算进度
                console.log('上传成功' + data.uploadStatus)
                self.percent = parseInt((data.size / self.fileSize) * 100)
                console.log(data.size)
                console.log('成功ID' + data.saveFileId)
                if (data.uploadStatus) {
                    self.status = '上传成功'
                }
            }, error: function (XMLHttpRequest, textStatus, errorThrown) {
                alert("服务器出错!");
            }
        });
    },
    uploadf() {
        let file = $("#fileupload")[0].files[0];  //文件对象
        this.isUpload(file)
    },
    }
}
</script>

<style>

</style>

