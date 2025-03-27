<template>
  <view class="cl-video-upload" :class="{ 'is-disabled': isDisabled }">
    <!-- 加载状态 -->
    <cl-loading-mask :loading="loading" :text="uploadText" />

    <!-- 视频列表 -->
    <view class="video-list">
      <view 
        class="video-item"
        v-for="(item, index) in videoList" 
      >
        <video
          class="video-player" style="width:200px;height:150px;"
          :src="item.url"
          :controls="false"
          :show-play-btn="false"
          :show-center-play-btn="true"
          objectFit="cover" 
        />
        
        
        
        <!-- 删除按钮 -->
        <view class="delete-btn" @click.stop="removeVideo(index)">
          <u-icon name="close-circle-fill" color="#ea424f" size="50"></u-icon>
        </view>
      </view>

      <!-- 添加按钮 -->
      <view 
        class="add-btn" 
        v-if="showAddButton"
        :style="videoItemStyle"
        @click="selectVideo"
      >
        <u-icon name="plus" color="#999" size="32"></u-icon>
        <text class="add-text">添加视频</text>
      </view>
    </view>
  </view>
</template>

<script>
import { isArray, isString } from "../../utils";
import Form from "../../mixins/form";

export default {
  name: "cl-video-upload",
  mixins: [Form],

  props: {
    value: [String, Array, Object],
    action: String,
    headers: Object,
    data: Object,
    name: {
      type: String,
      default: "file"
    },
    height: {
      type: String,
      default: "225rpx" // 4:3比例的高度 (300 * 3/4)
    },
    limit: {
      type: Number,
      default: 1
    },
    yii: {
      type: Boolean,
      default: false
    },
    // 其他props...
  },

  data() {
    return {
      loading: false,
      uploadText: "上传中...",
      videoList: []
    };
  },

  computed: {
    videoItemStyle() {
      return {
        height: this.height,
        width: `calc(${this.height} * 4 / 3)` // 4:3宽度
      };
    },

    showAddButton() {
      return this.videoList.length < this.limit && !this.isDisabled;
    }
  },

  watch: {
    value: {
      immediate: true,
      handler(val) {
        if (!val) {
          this.videoList = [];
        } else if (isArray(val)) {
          this.videoList = val.map(item => typeof item === 'string' ? { url: item } : item);
        } else if (isString(val)) {
          this.videoList = val.split(",").filter(Boolean).map(url => ({ url }));
        } else if (val.url) {
          this.videoList = [val];
        }
      }
    }
  },

  methods: {
    // 选择视频
    selectVideo() {
      if (this.isDisabled) return;

      uni.chooseVideo({
        sourceType: ['album', 'camera'],
        compressed: true,
        success: res => {
          this.uploadVideo(res.tempFilePath);
        }
      });
    },

    // 上传视频（使用您指定的uni.uploadFile方式）
    uploadVideo(filePath) {
      this.loading = true;
      this.uploadText = "视频上传中...";

      uni.uploadFile({
        url: this.action,
        filePath: filePath,
        name: this.name,
        header: this.headers,
        formData: this.data,
        success: (res) => {
          if(this.yii){
            let data = JSON.parse(res.data);
            this.addVideo(data.http_url);
            this.$emit("success", data);
          }else{
            const { data } = JSON.parse(res.data);
            this.addVideo(data.url || data);
            this.$emit("success", data);
          }  
        },
        fail: (err) => {
          this.$emit("error", err);
          uni.showToast({
            title: '上传失败',
            icon: 'none'
          });
        },
        complete: () => {
          this.loading = false;
        }
      });
    },

    // 添加视频到列表
    addVideo(url) {
      this.videoList.push({ url });
      this.emitChange();
    },

    // 预览视频
    previewVideo(url) {
      if (this.isDisabled) return;
      
      uni.previewMedia({
        sources: [{
          url: url,
          type: 'video'
        }]
      });
    },

    // 删除视频
    removeVideo(index) {
      if (this.isDisabled) return;
      
      this.videoList.splice(index, 1);
      this.emitChange();
      this.$emit("remove", index);
    },

    // 触发change事件
    emitChange() {
      let value = this.limit === 1 ? 
        (this.videoList[0] || '') : 
        this.videoList;
      
      this.$emit('input', value);
      this.$emit('change', value);
    }
  }
};
</script>

<style lang="scss" scoped>
.cl-video-upload {
  .video-list {
    display: flex;
    flex-wrap: wrap;
    gap: 15rpx;
  }

  .video-item, .add-btn {
    position: relative;
    background-color: #f5f5f5;
    border-radius: 8rpx;
    overflow: hidden;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .video-player {
    width: 100%;
    height: 100%;
  }
 

  .delete-btn {
    position: absolute;
    top: 8rpx;
    right: 8rpx;
    z-index: 3;
    background: rgba(255, 255, 255, 0.8);
    border-radius: 50%;
  }

  .add-btn {
    flex-direction: column;
    border: 1px dashed #ccc;
    
    .add-text {
      margin-top: 10rpx;
      font-size: 24rpx;
      color: #999;
    }
  }

  &.is-disabled {
    opacity: 0.6;
    
    .delete-btn {
      display: none;
    }
  }
}
</style>