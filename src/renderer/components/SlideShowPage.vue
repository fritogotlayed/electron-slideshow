<template>
  <div class="image">
    <div class="wrap">
      <p class="message" v-if="image === null">Images loading. Please wait.</p>
      <img v-if="image !== null" :src="dataUrl(image)" />
    </div>
  </div>
</template>

<script>
  let fs = require('fs')
  let path = require('path')
  const {webFrame} = require('electron')

  export default {
    name: 'slideshow',
    data () {
      return {
        validExtensions: ['.jpg', '.jpeg', '.gif', '.png', '.tif', '.tiff', '.bmp'],
        fileList: [],
        lastImages: [],
        image: null,
        timer: null
      }
    },
    methods: {
      _validFileExtension: function (fullPath) {
        return this.validExtensions.indexOf(path.extname(fullPath).toLowerCase()) > -1
      },
      dataUrl: function (buffer) {
        return 'data:image/jpeg;base64,' + btoa(
          new Uint8Array(buffer).reduce((data, byte) => data + String.fromCharCode(byte), '')
        )
      },
      loadFile: function (fullPath) {
        console.debug('Loading file: ' + fullPath)
        let parent = this
        fs.readFile(fullPath, function (err, data) {
          if (err) {
            alert('An error ocurred reading the file :' + err.message)
            return
          }
          parent.image = data
        })
      },
      loadFiles: function (dirPath) {
        let parent = this

        return new Promise((resolve, reject) => {
          fs.readdir(dirPath, function (err, files) {
            if (err) {
              alert('An error ocurred reading the file :' + err.message)
              reject(err.message)
            }

            let children = []
            let items = []

            files.forEach((file) => {
              let fullPath = path.join(dirPath, file)
              let fStat = fs.statSync(fullPath)

              if (fStat.isDirectory()) {
                children.push(parent.loadFiles(fullPath))
              } else {
                if (parent._validFileExtension(fullPath)) {
                  items.push(fullPath)
                } else {
                  console.log('Invalid extension: ' + fullPath)
                }
              }
            }) // files.forEach

            // Now that things are done merge our children
            if (children.length > 0) {
              Promise.all(children).then((value) => {
                value.forEach((list) => {
                  list.forEach((item) => {
                    items.push(item)
                    resolve(items)
                  })
                })
              })
            } else {
              resolve(items)
            }
          }) // fs.readdir
        }) // Promise
      },
      startTimer: function () {
        let parent = this

        let index = Math.floor(Math.random() * parent.fileList.length)
        let filePath = parent.fileList[index]

        if (parent.lastImages.length >= 10) {
          parent.lastImages.shift()
        }
        parent.lastImages.push(filePath)

        // remove the one we picked from the list
        parent.fileList.splice(index, 1)

        parent.loadFile(filePath)
        webFrame.clearCache()

        parent.timer = setTimeout(() => {
          parent.startTimer()
        }, 10 * 1000)
      }
    },
    mounted: function () {
      let parent = this
      let stamp1 = new Date()
      this.loadFiles('/home/malline/Pictures/Wallpapers').then((files) => {
        let stamp2 = new Date()
        console.log('Image load took: ' + (stamp2 - stamp1) / 1000 + ' seconds')
        parent.fileList = files
        parent.startTimer()
      })
    }
  }
</script>

<style>

  html, body, .image, .wrap {
    height: 100%;
    width: 100%;
    padding: 0;
    margin: 0;
    background-color: #232324;
  }

  .wrap {
    position: relative;
  }

  img {
    max-height: 100%;
    max-width: 100%;
    margin: auto;
    position: fixed;
    object-fit: contain;
    top: 0;
    left: 0;
    bottom: 0;
    right: 0;
    opacity: 1;
  }

  .message {
    text-align: center;
    color: antiquewhite;
  }

</style>
