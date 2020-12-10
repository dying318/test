<template>
  <div class="app-container">
    <div class="filter-container">
      <el-input
        v-model="listQuery.keyword"
        placeholder="关键词"
        style="width: 200px;"
        class="filter-item"
        clearable
        @keyup.enter.native="handleFilter"
      />
      <el-select v-model="listQuery.onSale" clearable placeholder="售出状态" class="filter-item">
        <el-option label="已售出" value="0"></el-option>
        <el-option label="未售出" value="1"></el-option>
      </el-select>
      <el-select v-model="listQuery.series" clearable placeholder="规格" class="filter-item" filterable>
        <el-option v-for="(item, index) in series" :key="index" :label="item.series + `(${item.count})`" :value="item.series"></el-option>
      </el-select>
      <el-select v-model="listQuery.project" clearable placeholder="作者" class="filter-item" filterable>
        <el-option v-for="(item, index) in projects" :key="index" :label="item.project + `(${item.count})`" :value="item.project"></el-option>
      </el-select>
      <el-select v-model="listQuery.category1" clearable placeholder="分类" class="filter-item" filterable>
        <el-option v-for="(item, index) in categories[1]" :key="index" :label="item.category + `(${item.count})`" :value="item.category"></el-option>
      </el-select>
      <el-button
        v-waves
        class="filter-item"
        type="primary"
        icon="el-icon-search"
        @click="handleFilter"
      >搜索</el-button>
      <br>
      <el-button class="filter-item" type="primary" icon="el-icon-edit" @click="handleCreate">新增</el-button>
      <el-button class="filter-item" style="margin-left: 10px;" type="primary" icon="el-icon-receiving" @click="batchOffline(1)">
        批量下架
      </el-button>
      <el-button class="filter-item" style="margin-left: 10px;" type="primary" icon="el-icon-receiving" @click="batchOffline(0)">
        批量上架
      </el-button>
    </div>
    <el-table
      :key="tableKey"
      v-loading="status.listLoading"
      :data="list"
      border=""
      fit
      stripe
      highlight-current-row
      style="width: 100%;"
      @sort-change="sortChange"
      @selection-change="handleSelectionChange"
    >
      <el-table-column type="selection" fixed></el-table-column>
      <el-table-column
        label="ID"
        prop="id"
        sortable="custom"
        align="center"
        width="80"
        :class-name="getSortClass('id')"
      >
        <template slot-scope="{row}">
          <span>{{ row.id }}</span>
        </template>
      </el-table-column>
      <el-table-column label="标题" align="center" width="100">
        <template slot-scope="{row}">
          <span>{{ row.title }}</span>
        </template>
      </el-table-column>
      <el-table-column label="编号" align="center">
        <template slot-scope="{row}">
          <span>{{ row.number || '-' }}</span>
        </template>
      </el-table-column>
      <el-table-column label="分类" align="center">
        <template slot-scope="{row}">
          <span>{{ row.category1 || '-' }}</span>
        </template>
      </el-table-column>
      <el-table-column label="规格" align="center">
        <template slot-scope="{row}">
          <span>{{ row.series.join('，') || '-' }}</span>
        </template>
      </el-table-column>
      <el-table-column label="作者" align="center">
        <template slot-scope="{row}">
          <span class="link-type"
          v-clipboard:success="clipboardSuccess"
          v-clipboard:copy="'https://fh.shobo.cn/#/pages/fh/List?project=' + encodeURIComponent(row.project)">
            {{ row.project || '-' }}
          </span>
        </template>
      </el-table-column>
      <el-table-column label="描述" align="center" show-overflow-tooltip>
        <template slot-scope="{row}">
          <span>{{ row.intro }}</span>
        </template>
      </el-table-column>
      <el-table-column label="二维码" align="center" width="70">
        <template slot-scope="{row}">
          <el-tooltip placement="top" content="点击下载">
                <img style="width: 45px;" class="el-link"
                  @click="download($config.apiUrl  + '/api/admin/artShow/qrcode/' + row.id, row.number, row.title)"
                  :src="$config.apiUrl  + '/api/admin/artShow/qrcode/' + row.id"
                >
          </el-tooltip>
        </template>
      </el-table-column>
      <el-table-column label="查看/视频/播完" align="center">
        <template slot-scope="{row}">
          <span>{{ row.view }} / {{ row.video_play }} / {{ row.video_finish }}</span>
        </template>
      </el-table-column>

      <el-table-column label="图片" align="center" width="125">
        <template slot-scope="{row}">
          <el-image
            v-for="(pic, index) in row.pics"
            :key="index"
            style="width: 50px; height: 50px; object-fit: cover;" fit="cover" :src="$config.ossUrl+ pic  + '-m_300'"
            :preview-src-list="row.pics.map(x => $config.ossUrl + x)" lazy>
          </el-image>
          <span v-if="row.pics.length < 1">-</span>
        </template>
      </el-table-column>
      <el-table-column label="视频/音频" align="center" width="110">
        <template slot-scope="{row}">
          <template v-for="(video, index) in row.videos">
            <a :href="$config.ossUrl+ video.video" target="_blank">视频{{index+1}}</a>
            /
            <a :href="$config.ossUrl+ video.audio" target="_blank">音频{{index+1}}</a>
            <br>
          </template>
          <span v-if="row.videos && row.videos.length < 1">-</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" width="150" fixed="right">
        <template slot-scope="{row,$index}">
          <el-button style="margin-bottom: 5px;" v-if="!row.sale_out" type="success" size="mini" @click="setSaleOut(row, $index)">售出</el-button>
          <el-button style="margin-bottom: 5px;" v-if="row.sale_out" type="info" size="mini" @click="setSaleOut(row, $index)">已售出</el-button>
          <el-button style="margin-bottom: 5px;margin-left: 2px;" v-if="!row.offline" type="success" size="mini" @click="setOffline(row, $index)">下架</el-button>
          <el-button style="margin-bottom: 5px;margin-left: 2px;" v-if="row.offline" type="info" size="mini" @click="setOffline(row, $index)">已下架</el-button>
          <br>
          <el-button style="margin-left: 2px;" type="primary" size="mini" @click="handleUpdate(row)">修改</el-button>
          <el-button style="margin-left: 2px;" size="mini" type="danger" @click="confirmDelete(row, $index)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <pagination
      v-show="total>0"
      :total="total"
      :page.sync="listQuery.page"
      :limit.sync="listQuery.pageLength"
      :pageSizes="[10, 20, 50, 100, 200, 500]"
      @pagination="getList"
    />
    <el-dialog :title="dialog.titles[dialog.status]" :visible.sync="dialog.show">
      <el-form
        ref="dataForm"
        :rules="rules"
        :model="temp"
        label-position="left"
        label-width="90px"
        style="width: 400px; margin-left:50px;"
      >
        <el-form-item label="标题" prop="title">
          <el-input ref="title" v-model="temp.title"/>
        </el-form-item>
        <el-form-item label="编号" prop="number">
          <el-input v-model="temp.number"/>
        </el-form-item>
        <el-form-item label="价格">
          <el-input v-model="temp.price" type="number"/>
        </el-form-item>
        <el-form-item label="描述" prop="intro">
          <el-input v-model="temp.intro" :autosize="{ minRows: 2, maxRows: 4}" type="textarea"/>
        </el-form-item>
        <el-form-item label="作者">
          <el-input v-model="temp.project"/>

          <!-- <el-select v-model="temp.series" filterable multiple allow-create default-first-option>
            <el-option v-for="(series, index) in series" :key="index" :label="series.series + `(${series.count})`" :value="series.series"></el-option>
          </el-select> -->
        </el-form-item>
        <el-form-item label="规格">
          <el-input v-model="temp.series"/>

          <!-- <el-select v-model="temp.series" filterable multiple allow-create default-first-option>
            <el-option v-for="(series, index) in series" :key="index" :label="series.series + `(${series.count})`" :value="series.series"></el-option>
          </el-select> -->
        </el-form-item>
        <el-form-item label="大类" prop="category1">
          <el-select v-model="temp.category1" filterable allow-create default-first-option>
            <el-option v-for="(category, index) in categories[1]" :key="index" :label="category.category + `(${category.count})`" :value="category.category"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="上架">
          <el-radio v-model="temp.offline" :label="0">上架</el-radio>
          <el-radio v-model="temp.offline" :label="1">下架</el-radio>
        </el-form-item>
        <el-form-item label="图片" prop="pics">
          <el-upload
            ref="uploadPic"
            action="/api/admin/artShow/pic"
            :on-preview="handlePreview('pic')"
            :on-remove="handleRemove('pic')"
            :on-change="handleChange('pic')"
            :before-upload="beforeUpload"
            :before-remove="beforeRemove"
            :auto-upload="false"
            :file-list="picList"
            multiple
          >
            <el-button slot="trigger" size="small" type="primary">选择文件</el-button>
            <el-button size="small" type="success" @click="submitUpload('uploadPic')">上传到服务器</el-button>
          </el-upload>
          <div class="upload-files">
            <div v-for="(pic, index) in temp.pics" :key="index" class="upload-file">
              <img class="upload-image" :src="$config.ossUrl + pic">
              <i class="el-icon-close" @click="temp.pics.splice(index, 1)"></i>
            </div>
          </div>
        </el-form-item>
        <el-form-item label="视频" >
          <el-upload
            ref="uploadVideo"
            action="/api/admin/artShow/video"
            :on-preview="handlePreview('video')"
            :on-remove="handleRemove('video')"
            :on-change="handleChange('video')"
            :before-remove="beforeRemove"
            :auto-upload="false"
            :file-list="videoList"
            multiple
          >
            <el-button size="small" slot="trigger" type="primary">选择文件</el-button>
            <el-button size="small" type="success" @click="submitUpload('uploadVideo')">上传到服务器</el-button>
          </el-upload>
          <div class="upload-files">
            <div v-for="(video, index) in temp.videos" :key="index" class="upload-file">
              <video :key="index"
                v-if="video.video"
                class="upload-video"
                :src="$config.ossUrl + video.video"
                controls
              />
              <i class="el-icon-close" @click="temp.videos.splice(index, 1)"></i>
            </div>
          </div>
            <!-- <audio
              v-if="video.audio"
              style="width: 138px;"
              :src="$config.ossUrl + video.audio"
              controls
            /> -->
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-checkbox
          v-if="dialog.status === 'create'"
          v-model="dialog.notClose"
          style="margin-right: 20px"
        >继续添加</el-checkbox>
        <el-button @click="dialog.show = false">取消</el-button>
        <el-button type="primary" @click="confirmDialog">确定</el-button>
      </div>
    </el-dialog>
    <el-tooltip placement="top" content="回到顶部">
      <back-to-top :visibility-height="1200" :back-position="50" transition-name="fade"/>
    </el-tooltip>
    <div id="view"></div>
  </div>
</template>

<script>
import {
  fetchList,
  fetchSeries,
  fetchProjects,
  fetchCategories,
  createArtShow,
  updateArtShow,
  batchUpdateArtShow,
  deleteArtShow
} from '@/api/artShow'
import waves from '@/directive/waves' // waves directive
import Pagination from '@/components/Pagination' // secondary package based on el-pagination
import BackToTop from '@/components/BackToTop'

import clipboard from '@/directive/clipboard/index.js' // use clipboard by v-directive

const temp = () => {
  return {
    id: undefined,
    title: undefined,
    number: undefined,
    intro: undefined,
    series: undefined,
    category1: undefined,
    category2: undefined,
    category3: undefined,
    offline: "0",
    stock: 1,
    pics: [],
    videos: [],
    video: undefined,
    audio: undefined,
    duration: undefined,
    poster: undefined
  }
}

export default {
  components: { Pagination, BackToTop },
  directives: { waves, clipboard },
  filters: {},
  data() {
    return {
      listQuery: {
        page: 1,
        pageLength: 20,
        keyword: '',
        category1: '',
        category2: '',
        category3: '',
        sort: 'id|desc'
      },
      status: {
        listLoading: false
      },
      list: [],
      series: [],
      projects: [],
      categories: {1:[], 2:[], 3:[]},
      temp: temp(),
      dialog: {
        titles: { create: '新增', update: '编辑' },
        status: '',
        show: false,
        notClose: false
      },
      tableKey: 0,
      total: 0,
      rules: {
        title: [{ required: true, message: '请填写标题', trigger: 'change' }],
        intro: [{ required: true, message: '请填写描述', trigger: 'change' }],
        pics: [{ required: true, message: '请上传图片', trigger: 'change' }],
        video: [{ required: true, message: '请上传视频', trigger: 'change' }],
        category1: [{ required: true, message: '请选择分类', trigger: 'change' }],
      },
      picList: [],
      videoList: []
    }
  },
  computed: {
  },
  watch: {
    showColumn: {
      handler: function() {
        this.tableKey += 1
        localStorage.setItem('showColumn', JSON.stringify(this.showColumn))
      },
      deep: true
    },
    picList: {
      handler: function() {
        const pics = []
        this.picList.forEach(pic => {
          if (pic.response && pic.response.data && !pic.finished) {
            pic.finished = true
            pics.push(pic.response.data)
          }
        })
        this.temp.pics = this.temp.pics.concat(pics)
      },
      deep: true
    },
    videoList: {
      handler: function() {
        // if (this.videoList[0].response) {
        //   this.temp.video = this.videoList[0].response.data.video
        //   this.temp.audio = this.videoList[0].response.data.audio
        //   this.temp.duration = this.videoList[0].response.data.duration
        //   this.temp.poster = this.videoList[0].response.data.poster
        // }
        const videos = []
        this.videoList.forEach(video => {
          if (video.response && video.response.data.video && !video.finished) {
            video.finished = true
            videos.push({
              video: video.response.data.video,
              audio: video.response.data.audio,
              duration: video.response.data.duration,
              poster: video.response.data.poster
            })
          }
        })

        this.temp.videos = this.temp.videos.concat(videos)
      },
      deep: true
    }
  },
  created() {
    this.getList()
    this.getSeries()
    this.getProjects()
    this.getCategories(1)
    this.getCategories(2)
    this.getCategories(3)
  },
  methods: {
    download(src, number, title) {
      var imgtwo = new Image()
      imgtwo.crossOrigin = 'Anonymous'
      imgtwo.src = src
      imgtwo.onload = () => {
        let height = 500
        if (number) height = 550
        const screenshotCanvas = document.createElement('canvas')
        screenshotCanvas.width = 500
        screenshotCanvas.height = height
        const ctx = screenshotCanvas.getContext('2d')
        ctx.fillStyle = '#fff'
        ctx.fillRect(0, 0, 500, height)
        ctx.drawImage(imgtwo, 0, 0, 500, 500)

        if (number) {
          ctx.fillStyle = '#333'
          ctx.textAlign = 'center'
          ctx.font = '22px "微软雅黑"'
          ctx.textAlign = 'center'
          ctx.textBaseline = 'top'
          ctx.fillText(number, 250, 500)
        }
        const alink = document.createElement('a')
        alink.href = screenshotCanvas.toDataURL('image/png')
        alink.download = `${(number && number + '-' + title) || title}.png`
        alink.click()
      }
    },
    clipboardSuccess() {
      this.$message({
        message: '已复制链接',
        type: 'success',
        duration: 1500
      })
    },
    getList() {
      this.status.listLoading = true
      fetchList(this.listQuery).then(response => {
        this.status.listLoading = false
        this.list = response.data.data
        this.total = response.data.total
      })
    },
    getSeries() {
      fetchSeries().then(response => {
        this.series = response.data
      })
    },
    getProjects() {
      fetchProjects().then(response => {
        this.projects = response.data
      })
    },
    getCategories(type) {
      fetchCategories(type).then(response => {
        this.categories[type] = response.data
      })
    },
    getSortClass: function(key) {
      const sort = this.listQuery.sort
      return sort === `${key}|asc` ? 'ascending' : 'descending'
    },
    handleRemove(type) {
      return (file, fileList) => {}
    },
    handlePreview(type) {
      return file => {
        if (type === 'pic') {
        } else {
        }
      }
    },
    handleChange(type) {
      return (file, fileList) => {
        if (fileList.length > 0) {
          if (type === 'video') {
            this.videoList = fileList
          } else {
            this.picList = fileList
          }
        }
      }
    },
    beforeUpload(file) {
      const sizeLimit = file.size / 1024 / 1024 < 15;

      if (!sizeLimit) {
        this.$message.error('图片大小不能超过 15MB!');
      }

      return sizeLimit
    },
    beforeRemove(file, fileList) {
      if (this.beforeUpload(file)) {
        return this.$confirm(`确定移除 ${file.name}？`)
      } else {
        return false
      }
    },
    submitUpload(ref) {
      this.$refs[ref].submit()
    },
    confirmDialog() {
      switch (this.dialog.status) {
        case 'create':
          this.createArtShow()
          break
        case 'update':
          this.updateArtShow()
          break
      }
    },
    resetTemp() {
      this.picList = this.videoList = []
      const offline = this.temp.offline
      this.temp = temp()
      this.temp.offline = offline
    },
    createArtShow() {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          createArtShow(this.temp).then(result => {
            this.list.unshift(result.data)
            this.resetTemp()
            this.dialog.show = this.dialog.notClose
            this.$notify({
              title: '添加成功',
              message: '添加成功',
              type: 'success',
              duration: 2000
            })
          })
        }
      })
    },
    updateArtShow() {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          if (typeof this.temp.series === 'string') {
            this.temp.series = [this.temp.series]
          }
          const tempData = Object.assign({}, this.temp)
          updateArtShow(tempData).then(() => {
            const index = this.list.findIndex(v => v.id === this.temp.id)
            this.list.splice(index, 1, this.temp)
            this.dialog.show = false
            this.$notify({
              title: '更新成功',
              message: '更新成功',
              type: 'success',
              duration: 2000
            })
          })
        }
      })
    },
    sortChange(data) {
      const { prop, order } = data
      this.sortBy(prop, order)
    },
    sortBy(field, order) {
      if (order === 'ascending') {
        this.listQuery.sort = field + '|asc'
      } else {
        this.listQuery.sort = field + '|desc'
      }
      this.handleFilter()
    },
    handleFilter() {
      this.listQuery.page = 1
      this.getList()
    },
    handleCreate() {
      this.resetTemp()
      this.dialog.status = 'create'
      this.dialog.show = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
        this.$refs['title'].focus()
      })
    },
    handleUpdate(row) {
      this.temp = Object.assign({}, row) // copy obj
      this.dialog.status = 'update'
      this.dialog.show = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    confirmDelete(row, index) {
      this.$confirm('确认删除?').then(() => {
        deleteArtShow(row.id).then(result => {
          this.list.splice(index, 1)
        })
      })
    },
    setSaleOut(row, index) {
      row.sale_out = row.sale_out ? 0 : 1
      updateArtShow(row).then(() => {
        this.list.splice(index, 1, row)
      })
    },
    setOffline(row, index) {
      row.offline = row.offline ? 0 : 1
      updateArtShow(row).then(() => {
        this.list.splice(index, 1, row)
      })
    },
    batchOffline(offline) {
      this.$confirm(`确认${offline ? '下架' : '上架'}?`).then(() => {
        batchUpdateArtShow({ids: this.selectedIds, offline: offline}).then(result => {
          this.$notify({
            title: '更新成功',
            message: '批量更新成功',
            type: 'success',
            duration: 2000
          })
          this.selectedIds.forEach(id => {
            const index = this.list.findIndex(v => v.id === id)
            this.list.splice(index, 1, Object.assign(this.list[index], {offline: offline}))
          })
        })
      }).catch(error => {
        this.$notify({
          title: '更新失败',
          message: '批量更新失败',
          type: 'success',
          duration: 2000
        })
      })
    },
    handleSelectionChange(rows) {
      this.selectedIds = rows.map(x => x.id)
    },
  }
}
</script>

<style lang="scss" scoped>
.line {
  text-align: center;
}
.price-divider {
  margin: 12px 0;
}

.upload-files {
  width:400px;
  display: flex;
  flex-wrap: wrap;
  .upload-file {
    margin-top: 10px;
    margin-right: 10px;
    display: flex;
    flex-direction: column;
    align-items: center;
    border: 1px solid #ccc;
    box-shadow: 0px 0px 10px 3px #ccc;

    .upload-image {
      width: 120px; height: 120px; object-fit: contain;
    }
    .upload-video {
      width: 120px; height: 120px;
    }
    .el-icon-close {
      font-size: 22px;
      color: red;
    }
  }
}
</style>
