<template lang="jade">
.admin#add-post
  .title
    h1 {{$route.meta.title}}
  el-form(ref='add-post-form', :model='form', label-position='top', :rules="rules")
    el-form-item(label='标题', prop='title')
      el-input(v-model='form.title')
    el-form-item(label='摘要', prop='abstract')
      el-input(type='textarea',v-model='form.abstract')
    el-form-item(label='CCID')
      el-input(placeholder='视频类文章填写该项', v-model='form.video_id')
    el-form-item(label='正文', prop='content')
      veditor
    el-form-item(label='添加标签（标签至少3个）', prop='tags')
      search-tag(:callback='searchTag', :tags='form.tags')
    el-form-item(label='栏目选择', prop='column_id')
      search-column(:callback='searchColumn', :column='form.column')
    el-form-item(label='文章头图（1400×780）', prop='cover_id')
      upload(:callback='uploadImage', :url='form.cover_url', :uploadDelete="uploadDelete")
    el-form-item(label='作者', prop='authors')
      search-user(:callback='searchUser', :authors='form.authors_full',:multiple='true')
    el-form-item(label='状态')
      el-select(v-model='form.state', placeholder='请选择')
        el-option(v-for='item in post_states',
                  :label='item.title',
                  :value='item.val',
                  :key='item.val')
    el-form-item(label='发布时间', prop='date', v-if='form.state === "published"')
      el-date-picker(v-model='form.auto_publish_at',
                     type='datetime',
                     placeholder='选择日期时间')
    el-form-item(label='')
      el-button(type='primary', :disabled='disabled', @click='submitForm') 提交
      el-button(type='danger', @click='close') 关闭
</template>

<script>
import api from 'stores/api'

export default {
  data () {
    const validateContent = (rule, value, callback) => {
      getContent(this)
      if (this.form.content_source === '') {
        callback(new Error('请输入内容'))
      } else {
        callback()
      }
    }
    const validateArray = (rule, value, callback) => {
      if (value === undefined || value.length && value.length === 0) {
        callback(new Error('请输入内容'))
      } else {
        callback()
      }
    }
    return {
      disabled: false,
      form: {
        title: '',
        abstract: '',
        content_type: 'html',
        content_source: '',
        tags: [],
        column: [],
        column_id: '',
        cover_id: '',
        cover_url: '',
        authors: [],
        authors_full: [],
        auto_publish_at: '',
        state: 'unpublished',
        video_id: '',
        post_type: 'text'
      },
      rules: {
        title: [
          { required: true, message: '请输入文章标题', trigger: 'blur', min: 0 }
        ],
        abstract: [
          { required: true, message: '请输入文章摘要', trigger: 'blur', min: 0 }
        ],
        content: [
          { required: true, validator: validateContent, trigger: 'blur' }
        ],
        date: [
          { type: 'date', message: '请选择日期', trigger: 'change' }
        ],
        column_id: [
          { required: true, validator: validateArray, message: '请至少选择一个专栏', trigger: 'change' }
        ],
        tags: [
          { required: true, validator: validateArray, message: '请至少选择一个标签', trigger: 'change' }
        ],
        cover_id: [
          { type: 'number', required: true, message: '请上传文章头图', trigger: 'change' }
        ],
        authors: [
          { required: true, validator: validateArray, message: '请至少选择一个作者', trigger: 'change' }
        ]
      },
      post_states: [{
        title: '草稿',
        val: 'unpublished'
      }, {
        title: '已发布',
        val: 'published'
      }],
      content_types: [{
        title: '富文本',
        val: 'html'
      }, {
        title: 'markdown',
        val: 'markdown'
      }]
    }
  },
  methods: {
    submitForm () {
      this.$refs['add-post-form'].validate((valid) => {
        if (valid) {
          if (this.form.video_id !== '') {
            this.form.post_type = 'video'
          }
          if (this.form.auto_publish_at === '') {
            delete this.form.auto_publish_at
          }
          if (this.$route.query.id) {
            updatePost(this)
          } else {
            createPost(this)
          }
        } else {
          this.$message.error('内容信息不完整, 请完善后再提交!')
          return false
        }
      })
    },
    resetForm () {
      this.$refs['add-post-form'].resetFields()
    },
    uploadImage (img) {
      this.form.cover_id = img.id
      this.form.cover_url = ''
    },
    uploadDelete () {
      this.form.cover_url = 'deleted'
      this.form.cover_id = ''
    },
    searchUser (users) {
      this.form.authors = users
    },
    searchTag (tags) {
      this.form.tags = tags
    },
    searchColumn (column) {
      this.form.column_id = column
    },
    close () {
      this.$router.push('/posts')
    }
  },
  watch: {
    'form.content_type': function (val) {
      const id = this.$route.query.id ? `&id=${this.$route.query.id}` : ''
      this.$router.push(`${this.$route.path}?content_type=${val}${id}`)
      addContent(this, val)
    }
  },
  mounted () {
    if (this.$route.query.id) { getPost(this) }
  }
}

function getContent (_this) {
  _this.form.content_source = _this.$store.state.htmlEditor.txt.html()
}

function addContent (_this, val) {
  setTimeout(() => {
    _this.$store.state.htmlEditor.txt.html(_this.form.content_source)
  }, 100)
}

function updatePost (_this) {
  getContent(_this)
  _this.disabled = true
  api.put(`admin/posts/${_this.$route.query.id}`, _this.form)
  .then((result) => {
    _this.$message.success('success')
    _this.$router.push(`/posts?state=${_this.form.state}`)
  }).catch((err) => {
    _this.disabled = false
    _this.$message.error(err.toString())
  })
}

function createPost (_this) {
  getContent(_this)
  _this.disabled = true
  api.post('admin/posts', _this.form)
  .then((result) => {
    console.log(result)
    _this.$message.success('success')
    _this.$router.push(`/posts?state=${_this.form.state}`)
  }).catch((err) => {
    _this.disabled = false
    _this.$message.error(err.toString())
  })
}

function getPost (_this) {
  api.get(`admin/posts/${_this.$route.query.id}`)
  .then((result) => {
    const {post} = result.data
    post.column_id = post.column.id
    Object.keys(_this.form).forEach(key => {
      _this.form[key] = post[key]
    })
    _this.form.auto_publish_at = post.published_at
    _this.form.authors_full = post.authors
    _this.form.authors = _this.form.authors_full ? _this.form.authors_full.map(el => el.id) : []
    _this.form.video_id = post.extra.video_id
    addContent(_this, _this.form.content_type)
  }).catch((err) => {
    _this.$message.error(err.toString())
  })
}
</script>

<style lang="stylus" scoped>
.el-input, .el-textarea
  width 50%
.el-select-dropdown
  z-index 99999 !important

#add-post
  hegiht auto !important
</style>
