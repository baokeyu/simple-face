<template>
  <div class="hello">
    <div>
      <div @click="click">拍照</div>
      <input @change="hanlder" type="file" style="visibility:hidden" ref="photo" accept="image/*" capture='camera'>
    </div>
    <img ref="display" style="width: 100%;" />
    <canvas ref="canvas" />
  </div>
</template>

<script>
import OSS from 'ali-oss';
import Conf from '@/conf';
import EXIF from 'exif-js';

export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  data() {
    return {
      title: "exif"
    }
  },
  created() {
    this.client = new OSS({ 
      accessKeyId: Conf.ak.accessKeyId,
      accessKeySecret: Conf.ak.accessKeySecret,
      stsToken: Conf.ak.stsToken,
      region: Conf.oss.region,
      bucket: Conf.oss.bucket
    });
  },
  methods: {
    click() {
      this.$refs.value = "";
      this.$refs.photo.click();
    },
    format(process) {
      if(typeof process === "number") {
        return Math.floor(process * 100);
      }
      return 0;
    },
    async hanlder(e) {
      if(!e.currentTarget.files || e.currentTarget.files.length == 0) {
        return ;
      }
      var self = this;
      var file = e.currentTarget.files[0];
      var ext = file.name.split(".")[1];
      var key = `testbky/face/${new Date().getTime() + Math.random()}.${ext}`;
      this.show(file);
      // 如果图片大小小于400kb，则直接上传
      if(file.size < 1024 * 400) {
        await this.searchFace(key, file);
        return ;
      }
      var compressFile = await this.compress(file);
      await this.searchFace(key, compressFile);
    },
    async searchFace(key, file) {
      await this.upload(key, file);
    },
    download(file) {
      var reader = new FileReader();
      reader.readAsDataURL(file);   // 转换为base64，可以直接放入a标签href
      reader.onload = function (e) {
        var a = document.createElement('a');   // 转换完成，创建一个a标签用于下载
        a.download = 'aaa.jpg';
        a.href = e.target.result;
        document.body.append(a);
        a.click();
      }
    },
    upload(key, file) {
      return new Promise((resolve, reject) => {
        this.client.multipartUpload(key, file, {
          parallel: 4,
          partSize: 1024 * 1024, //1024 * 1024,
          mime: file.type
        }).then(async (result) => {
          resolve();
        }).catch((err) => {
          reject();
        });
      });
    },
    compress(file) {
      var self = this;
      return new Promise((resolve, reject) => {
        var reader = new FileReader();
        var img = new Image();
        reader.onload = function () {
          img.src = this.result;
          img.onload = function () {
            EXIF.getData(this, function() {
              var orientation = EXIF.getTag(this, "Orientation");
              var K = img.width / Conf.setting.imageWidth;
              var canvasWidth = Conf.setting.imageWidth;
              var canvasHeight = Math.floor(img.height / K);
              var canvas = document.createElement("canvas");
              
              canvas.width = canvasWidth;
              canvas.height = canvasHeight;
              
              if(orientation && orientation == 6) {
                canvas.width = canvasHeight;
                canvas.height = canvasWidth;
                canvas.getContext("2d").rotate(Math.PI / 2);
                canvas.getContext("2d").drawImage(this, 0, -canvasHeight, canvasWidth, canvasHeight);
              }
              else if(orientation && orientation == 3) {
                canvas.width = canvasWidth;
                canvas.height = canvasHeight;
                canvas.getContext("2d").rotate(Math.PI);
                canvas.getContext("2d").drawImage(this, -canvasWidth, -canvasHeight, canvasWidth, canvasHeight);
              }
              else if(orientation && orientation == 8) {
                canvas.width = canvasHeight;
                canvas.height = canvasWidth;
                canvas.getContext("2d").rotate(3 * Math.PI / 2);
                canvas.getContext("2d").drawImage(this, -canvasWidth, 0, canvasWidth, canvasHeight);
              }
              else {
                canvas.width = canvasWidth;
                canvas.height = canvasHeight;
                canvas.getContext("2d").drawImage(img, 0, 0, canvasWidth, canvasHeight);
              }
              canvas.toBlob(async function(blob) {
                resolve(blob);
              }, file.type);
            });
          };
        };
        reader.readAsDataURL(file);
      });
    },
    show(file) {
      var self = this;
      var reader = new FileReader();
      reader.onload = function () {
        self.$refs.display.src = this.result;
      };
      reader.readAsDataURL(file);
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
