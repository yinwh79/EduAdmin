
<template>
  <!-- --------------------学员的跟进记录模块----------------------- -->
  <div class="p_both10">
    <el-table :data="customBuyCourseList" border tooltip-effect="dark" style="width: 100%">
      <el-table-column prop="CourseImage" width="90" label="课程logo">
        <template slot-scope="scope">
          <div
            class="coverImg wid80 hgt80"
            :style="{'backgroundImage':'url('+scope.row.CourseImage+')'}"
          />
        </template>
      </el-table-column>
      <el-table-column prop="Label" label="课程名称" :show-overflow-tooltip="true" />
      <el-table-column prop="OldPayment" label="定价" width="80" />
      <el-table-column prop="Payment" label="实付" width="80" />
      <el-table-column prop="Createtime" :formatter="dateFormat" label="购买时间" width="90" />
      <el-table-column prop="StartTime" :formatter="dateFormat" label="生效日期" width="90" />
      <el-table-column prop="EndTime" :formatter="dateFormat" label="结束日期" width="90" />
      <el-table-column
        prop="PayMethod"
        :formatter="paymethodFormat"
        label="支付方式"
        :show-overflow-tooltip="true"
        width="120"
      />
      <el-table-column label="附加设置" width="110">
        <template slot-scope="scope">
          <el-checkbox
            :checked="scope.row.ForbiddenVideo"
            @change="checked=>onChangeExamHandler(checked,scope.row,1)"
          >禁止网课</el-checkbox>
        </template>
      </el-table-column>
      <!-- <el-table-column label="操作" width="100" fixed="right">
        <template slot-scope="scope">
          <el-button type="primary" @click="deleteBuyCourseRecord(scope.row.Id,scope.$index)">删除</el-button>
        </template>
      </el-table-column>-->
    </el-table>
    <div>
      <el-button type="primary" class="m-v-15" @click="addBuyCourse()">手动添加购买记录</el-button>
    </div>
    <!-- 添加购买课程记录 -->
    <el-dialog
      :close-on-click-modal="false"
      :visible.sync="addBuyCourseDialog"
      :append-to-body="true"
      width="500px"
      title="添加购买记录"
    >
      <el-form
        ref="refbuyCourse"
        :model="addBuyCourseFormData"
        :rules="buyCourseRules"
        label-width="80px"
        style="padding:30px  "
        size="small"
        class="dialog-body-pad addBuyCourse"
      >
        <div class="between-center">
          <el-form-item label="学院">
            <el-select v-model="selectCollegeIndex" placeholder="请选择学院" @change="collegeChange">
              <el-option
                v-for="(item,index) in $store.getters.app.collegeWithCourseKind"
                :key="index"
                :label="item.Label"
                :value="index"
              />
            </el-select>
          </el-form-item>
          <el-form-item label="课程类别">
            <el-select
              v-model="selectCourseKindIndex"
              placeholder="请选择课程分类"
              @change="changeSelectCourseKind"
            >
              <el-option
                v-for="(item,index) in courseKindsOps"
                :key="index"
                :label="item.Label"
                :value="index"
              />
            </el-select>
          </el-form-item>
        </div>
        <el-form-item label="课程" prop="selectedCourseId">
          <el-select
            v-model="addBuyCourseFormData.selectedCourseId"
            placeholder="请选择课程"
            @change="forceUpdate"
          >
            <el-option
              v-for="item in courseOptions"
              :key="item.Id"
              :label="item.Label"
              :value="item.Id"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="课程时效" prop="courseStudyTime">
          <div class="between-center">
            <el-date-picker
              v-model="addBuyCourseFormData.courseStudyTime"
              type="daterange"
              align="right"
              style="width:250px"
              unlink-panels
              range-separator="至"
              start-placeholder="生效时间"
              end-placeholder="结束日期"
              value-format="timestamp"
              :picker-options="common.dateFuturePickerOptions"
              @change="changeSelectStudyTime"
            />
            <span class="m_both5">共{{ courseStudyDay }}天</span>
          </div>
        </el-form-item>
        <div class="between-center">
          <el-form-item label="实收金额" prop="actualPrice">
            <el-input-number
              v-model="addBuyCourseFormData.actualPrice"
              style="width:250px"
              :min="0"
            />
          </el-form-item>
          <el-checkbox v-model="sendSMS">短信通知</el-checkbox>
        </div>
      </el-form>
      <div class="around-center hgt60 bge0e3ea">
        <div>
          <el-button @click="addBuyCourseDialog = false">取 消</el-button>
          <el-button type="primary" class="m-l-40" @click="saveBuyCourseRecord">确 认</el-button>
        </div>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import {
  GetStudentDataTrackAnalysis,
  getCustomTracks,
  addcustomTracks,
  uploadImgInTracks,
  replyTracks,
  receiveSmsTrack,
  getTrackList,
  getCustomBuyCouseRecord,
  addCustomBuyCourseRecord,
  customAllowDoExercise,
  deleteBuyCourse,
  getStudentList,
  addStudent,
  editStudent,
  resetStudentPassword,
  setStudentStatus,
  checkTelephone,
  setStar,
  batchChangeManager,
  getStudentStatustByStudent
} from "@/api/student";

import $ImgAPI from "@/api/ImgAPI";
import { GetCourseOfKind } from "@/api/course";
import common from "@/utils/common";
export default {
  props: {
    // 学员数据
    customData: {
      type: Object,
      default: function() {
        return { Id: 0 };
      }
    }
  },
  name: "CustomBuyRecord",
  data() {
    return {
      common,
      sendSMS: true,
      // 学员Id
      customID: null,
      // 该学员所有的购买记录
      customBuyCourseList: [],
      // 添加购买记录弹出框
      addBuyCourseDialog: false,
      // 添加购买记录的表单数据
      addBuyCourseFormData: {},
      // 学习的时长-天
      courseStudyDay: 0,
      // 选中的课程类别ID的值
      selectCourseKindId: null,
      // 选中的课程类别Index的值
      selectCourseKindIndex: null,
      // 选中学院的Id
      selectCollegeid: null,
      // 选中学院的Index
      selectCollegeIndex: null,
      // 课程类别的选项列表
      courseKindsOps: [],
      // 课程的选项列表
      courseOptions: [],
      customItemData: {},
      // 表单验证
      buyCourseRules: {
        selectedCourseId: [
          { required: true, message: "课程不能为空", trigger: "blur" }
        ],
        courseStudyTime: [
          { required: true, message: "课时时效不能为空", trigger: "blur" }
        ],
        actualPrice: [
          { required: true, message: "实际金额不能为空", trigger: "blur" }
        ]
      }
    };
  },
 watch: {
    customData(newval) {
      this.fire();
    }
  }, 
 
  methods: {
     fire() {
    this.customBuyCourseList = [];
    this.customItemData = this.customData;
    this.getBuyCouseRecord();
  },
    // 根据选中的学院获取课程类别
    collegeChange(selVa) {
      this.selectCourseKindId = null;
      this.selectCourseKindIndex = null;
      this.addBuyCourseFormData.selectedCourseId = null;
      this.courseOptions = [];
      const courseKindArr = this.$store.getters.app.collegeWithCourseKind[selVa]
        .Children;
      this.courseKindsOps = courseKindArr || [];
      this.selectCollegeid = this.$store.getters.app.collegeWithCourseKind[
        selVa
      ].Id;
      if (this.courseKindsOps.length > 0) {
        this.selectCourseKindIndex = 0;
        this.changeSelectCourseKind(0);
      }
    },
    // 根据选中的课程类别获取课程
    async changeSelectCourseKind(selVa) {
      this.addBuyCourseFormData.selectedCourseId = null;
      this.selectCourseKindId = this.courseKindsOps[selVa].Id;
      const res = await GetCourseOfKind("", {
        kindid: this.selectCourseKindId
      });
      this.courseOptions = res.data ? res.data : [];
      if (this.courseOptions.length > 0) {
        this.addBuyCourseFormData.selectedCourseId = this.courseOptions[0].Id;
      }
    },
    forceUpdate($event) {
      this.$forceUpdate();
    },

    // 上传跟进记录的图片
    async uploadTrackImg(file) {
      let that = this;
      const res = await $ImgAPI.UploadImg("track", file.raw);
      if (res.code == 200) {
        that.$message({
          message: "操作成功",
          type: "success"
        });
        that.trackImgList.push(res.data);
      } else {
        that.$message({
          message: res.title,
          type: "warning"
        });
      }
    },

    // 获取学员的购买记录
    async getBuyCouseRecord() {
      if (this.customItemData.id && this.customItemData.id > 0) {
        const res = await getCustomBuyCouseRecord("", {
          studentid: this.customItemData.id,
          kind: 2,
          limit: 100000,
          offset: 0
        });
        if (res.data) {
          res.data.forEach(item => {
            const newitem = item;
            if (item.Info != "") {
              const info = JSON.parse(item.Info);
              newitem.ForbiddenVideo = info.ForbiddenVideo;
            }
            this.customBuyCourseList.unshift(newitem);
          });
        }
      }
    },
    // 点击添加购买课程记录
    addBuyCourse() {
      this.addBuyCourseDialog = true;
      this.addBuyCourseFormData = {};
    },

    // 显示日期处理
    dateFormat(row, column, cellValue, index) {
      return this.common.dateFormat(cellValue, false);
    },
    // 指定格式支付方式
    paymethodFormat(row, column, cellValue, index) {
      switch (cellValue) {
        case "":
          return "线下购买";
        case "alipay":
          return "支付宝";
        case "wxpay":
          return "微信";
      }
      return cellValue;
    },
    // 处理日期格式
    changeSelectStudyTime() {
      this.courseStudyDay =
        (this.addBuyCourseFormData.courseStudyTime[1] -
          this.addBuyCourseFormData.courseStudyTime[0]) /
        3600 /
        24000;
    },
    // 保存购买记录
    saveBuyCourseRecord() {
      this.$refs.refbuyCourse.validate(valid => {
        if (valid) {
          const that = this; 
          that.customBuyCourseList = [];
          that
            .$confirm("确认添加吗", "提示", {
              confirmButtonText: "确定",
              cancelButtonText: "取消",
              type: "warning"
            })
            .then(async () => {
              const params = {
                studentid: that.customItemData.id,
                starttime: parseInt(
                  that.addBuyCourseFormData.courseStudyTime[0] / 1000
                ),
                endtime: parseInt(
                  that.addBuyCourseFormData.courseStudyTime[1] / 1000
                ),
                shishou: that.addBuyCourseFormData.actualPrice,
                sms: that.sendSMS,
                collegeid: that.selectCollegeid
              };
              const res = await addCustomBuyCourseRecord(
                that.addBuyCourseFormData.selectedCourseId,
                params,
                ""
              );
              if (res.code == 200) {
                if (res.data) {
                  res.data.forEach(item => {
                    const newitem = item;
                    if (item.Info != "") {
                      const info = JSON.parse(item.Info);
                      newitem.ForbiddenVideo = info.ForbiddenVideo;
                    }
                    that.customBuyCourseList.unshift(newitem);
                  });
                }
                that.addBuyCourseDialog = false;
                if (that.sendSMS == true) {
                  this.$message({
                    message: "手动添加成功，并已发送短信告知对方",
                    type: "success"
                  });
                } else {
                  this.$message({
                    message: "手动添加成功 .没有用短信通知对方",
                    type: "success"
                  });
                }
              }
            })
            .catch(() => {});
        } else {
          return false;
        }
      });
    },
    // 允许做题与否
    async onChangeExamHandler(checked, row, vedioOrExam) {
      const urlParams = row.Id + "/" + vedioOrExam + "/" + checked;
      const res = customAllowDoExercise(urlParams);
      if (res.code == 200) {
      }
    },
    // 删除购买记录
    deleteBuyCourseRecord(id, index) {
      this.$confirm("确定删除吗?", "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(async () => {
          const res = await deleteBuyCourse(id, "", "");
          if (res.code == 200) {
            this.$message({
              message: "操作成功",
              type: "success"
            });
            this.customBuyCourseList.splice(index, 1);
          }
        })
        .catch(() => {
          return false;
        });
    }
  }
};
</script>
<style scoped>
.addBuyCourse >>> .el-select,
.addBuyCourse >>> .el-date-editor,
.addBuyCourse >>> .el-input-number--small {
  width: 100%;
}
</style>
