<template>
  <main>
    <h1>透かし入れ</h1>
    <section>
      <h2>透かし</h2>
      <el-upload
        action=""
        list-type="picture-card"
        :on-change="handleIconChange"
        :on-remove="handleIconChange"
        :limit="1"
        :drag="true"
        :auto-upload="false"
      >
        <i class="el-icon-plus"></i>
      </el-upload>
    </section>
    <section>
      <h2>対象の画像</h2>
      <el-upload
        action=""
        list-type="picture-card"
        :on-change="handleTargetChange"
        :on-remove="handleTargetChange"
        :multiple="true"
        :drag="true"
        :auto-upload="false"
      >
        <i class="el-icon-plus"></i>
      </el-upload>
    </section>
    <section>
      <h2>調整</h2>
      <article>
        <h3>透かしの位置</h3>
        <el-select v-model="watermark.pos" placeholder="Position">
          <el-option
            v-for="item in watermark.options.pos"
            :key="item.value"
            :label="item.label"
            :value="item.value"
          >
          </el-option>
        </el-select>
      </article>
      <article>
        <h3>余白</h3>
        <table>
          <tbody>
            <tr v-for="item in watermark.options.margin" :key="item.value">
              <th>{{ item.label }}</th>
              <td>
                <el-input-number
                  v-model="watermark.margin[item.value]"
                  :min="0"
                  controls-position="right"
                ></el-input-number>
              </td>
              <td>px</td>
            </tr>
          </tbody>
        </table>
      </article>
      <article>
        <h3>サイズ</h3>
        <table>
          <tbody>
            <tr v-for="item in watermark.options.size.val" :key="item.value">
              <th>{{ item.label }}</th>
              <td>
                <el-input-number
                  v-model="watermark.size.val[item.value]"
                  :min="1"
                  :max="100"
                  controls-position="right"
                ></el-input-number>
              </td>
              <td>%</td>
            </tr>
            <tr>
              <th>優先</th>
              <td>
                <el-select
                  v-model="watermark.size.use"
                  placeholder="Size calculation priority"
                >
                  <el-option
                    v-for="item in watermark.options.size.use"
                    :key="item.value"
                    :label="item.label"
                    :value="item.value"
                  >
                  </el-option>
                </el-select>
              </td>
              <td>を優先する</td>
            </tr>
          </tbody>
        </table>
      </article>
      <article>
        <h2>不透明度</h2>
        <div>
          <el-input-number
            v-model="watermark.opacity"
            :min="0"
            :max="100"
            controls-position="right"
          ></el-input-number>
          <span class="space-15"></span>
          %
        </div>
      </article>
    </section>
    <section>
      <h2>結果</h2>
      <el-button type="primary" round @click="draw">Generate</el-button>
      <article v-if="$device.isMobileOrTablet" class="res">
        <el-row class="res-row">
          <el-col v-for="(item, index) in images" :key="index" class="card">
            <el-card :body-style="{ padding: '0px' }">
              <img :src="item" class="image" />
            </el-card>
          </el-col>
        </el-row>
      </article>
      <div id="image"></div>
    </section>
  </main>
</template>

<script>
const konva = require('konva')
export default {
  data: () => ({
    images: [],
    data: [],
    icon: {},
    iconSelected: false,
    canvas: null,
    watermark: {
      pos: 3, // 0: top-left, 1: top-right, 2: bottom-left, 3: bottom-right
      margin: [25, 25, 25, 25], // 0: top, 1: bottom, 2: left, 3: right
      size: {
        val: [20, 20], // %, 0: width, 1: height
        use: 0 // 0: width, 1: height
      },
      opacity: 80, // %
      options: {
        pos: [
          { label: '左上', value: 0 },
          { label: '右上', value: 1 },
          { label: '左下', value: 2 },
          { label: '右下', value: 3 }
        ],
        margin: [
          { label: '上', value: 0 },
          { label: '下', value: 1 },
          { label: '左', value: 2 },
          { label: '右', value: 3 }
        ],
        size: {
          val: [{ label: '幅', value: 0 }, { label: '高さ', value: 1 }],
          use: [{ label: '幅', value: 0 }, { label: '高さ', value: 1 }]
        }
      }
    }
  }),
  mounted() {
    this.canvas = new konva.Stage({
      container: 'image',
      width: 0,
      height: 0
    })
  },
  methods: {
    handleTargetChange(file, fileList) {
      const urls = fileList.map(e => ({ url: e.url, name: e.raw.name }))
      this.data = urls
    },
    handleIconChange(file, fileList) {
      this.iconSelected = fileList.length > 0
      this.icon = file.url
    },
    downloadURI(uri, name) {
      const link = document.createElement('a')
      link.download = name
      link.href = uri
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
    },
    draw() {
      if (!this.icon || !this.data[0]) {
        return false
      }
      const canvas = this.canvas
      let imageLayer, watermarkLayer
      const _this = this
      this.images = []
      for (const item of _this.data) {
        new Promise(function(resolve, reject) {
          _this.canvas.destroy()
          imageLayer = new konva.Layer()
          watermarkLayer = new konva.Layer()
          _this.canvas = new konva.Stage({
            container: 'image',
            width: 0,
            height: 0
          })
          konva.Image.fromURL(item.url, function(image) {
            const img = image
            const rect = img.getSelfRect()
            canvas.size({ width: rect.width, height: rect.height })
            imageLayer.add(img)
            imageLayer.draw()
            resolve()
          })
        })
          .then(function() {
            konva.Image.fromURL(_this.icon, function(image) {
              const img = image
              let rect = img.getSelfRect()
              const full = canvas.size()
              if (_this.watermark.size.use === 0) {
                const width = full.width * (_this.watermark.size.val[0] / 100)
                img.size({
                  width: width,
                  height: rect.height * (width / rect.width)
                })
              } else {
                const height = full.height * (_this.watermark.size.val[1] / 100)
                img.size({
                  width: rect.width * (height / rect.height),
                  height: height
                })
              }
              rect = img.size()
              const pos = {
                x: _this.watermark.pos % 2 === 1 ? full.width - rect.width : 0,
                y: _this.watermark.pos >= 2 ? full.height - rect.height : 0
              }
              const pattern = [[0, 2], [0, 3], [1, 2], [1, 3]]
              const applyMargin = pattern[_this.watermark.pos]
              img.absolutePosition({
                x: pos.x,
                y: pos.y
              })
              if (pos.y === 0) {
                img.move({ x: 0, y: _this.watermark.margin[applyMargin[0]] })
              } else {
                img.move({ x: 0, y: -_this.watermark.margin[applyMargin[0]] })
              }
              if (pos.x === 0) {
                img.move({ x: _this.watermark.margin[applyMargin[1]], y: 0 })
              } else {
                img.move({ x: -_this.watermark.margin[applyMargin[1]], y: 0 })
              }

              img.opacity(_this.watermark.opacity / 100)
              watermarkLayer.add(img)
              watermarkLayer.draw()
              if (_this.$device.isDesktop) {
                _this.downloadURI(canvas.toDataURL(), item.name)
              } else {
                _this.images.push(canvas.toDataURL())
              }
            })
          })
          .then(function() {
            canvas.add(imageLayer)
            canvas.add(watermarkLayer)
          })
      }
    }
  }
}
</script>

<style>
main {
  padding: 15px 25px;
  text-align: center;
}
.el-upload--picture-card {
  width: 148px;
  height: 148px;
  border: 0;
}
.el-upload-dragger {
  border: 0;
  width: 100%;
  height: 100%;
  border: 1px dashed #c0ccda;
}
#image {
  display: none;
}
img {
  object-fit: contain;
  object-position: center;
}
table {
  border-collapse: separate;
  border-spacing: 15px;
}
table th,
table td {
  border-spacing: 0;
}
.space-15 {
  font-size: 0;
  line-height: 0;
  display: inline-block;
  width: 15px;
}
article {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
}
.image {
  max-width: 80vw;
  width: 300px;
}
.res {
  margin: 30px 0;
}
.res-row *:not(img) {
  width: fit-content;
}
.card {
  margin: 25px;
}
</style>
