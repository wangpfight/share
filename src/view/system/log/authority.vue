<template>
  <div>
    <div class="page-title">权限设置</div>
    <div class="main-container">
       <el-row class="mar-t-20">
         <el-col :span="8">
           <label>用户名称：</label>
           <span>{{data.userName}}</span>
         </el-col>
         <el-col :span="8">
           <label>API名称：</label>
           <span>{{data.apiName}}</span>
         </el-col>
         <el-col :span="8">
           <label>订阅时间：</label>
           <span>{{data.apiBoughtTime}}</span>
         </el-col>

       </el-row>
      <el-row class="mar-t-20">
        <el-col :span="8">
          <label>接口类型：</label>
          <span>{{data.apiMainTypeValue}}</span>
        </el-col>
        <el-col :span="8">
          <label>使用/订阅次数：</label>
          <span>{{data.apiCallTimes}} / {{data.apiBookedTimes}}</span>
        </el-col>
        <el-col :span="8">
          <label>状态：</label>
          <span>{{data.callTypeValue}}</span>
        </el-col>
      </el-row>

      <el-row class="mar-t-20">
        <el-radio v-model="data.permissionType" label="0">默认开通全部权限</el-radio>
        <el-radio v-model="data.permissionType" label="1">设置接口调用数据范围</el-radio>
      </el-row>

      <el-table :data="list" border v-show="data.permissionType == 1"  class="mar-t-20">
        <el-table-column label="参数名称" prop="title"  class-name="first-column">
          <template slot-scope="scope">
           <el-select v-model="scope.row.parameterName" size="mini" class="form-input" @change="changeParam(scope.row, scope.$index)">
             <el-option v-for="item in paramArray" :label="item.label" :key="item.value" :value="item.value"></el-option>
           </el-select>
          </template>
        </el-table-column>
        <el-table-column label="参数说明" prop="desc" >
          <template slot-scope="scope">
            {{descObj[scope.row.parameterName]}}
          </template>
        </el-table-column>
        <el-table-column label="数据范围设置" prop="applicant" >
          <template slot-scope="scope">
            <el-input size="mini" v-model="scope.row.parameterValue" v-if="typeObj[scope.row.parameterName] != 'DATE' && typeObj[scope.row.parameterName] != 'TIMESTAMP'" maxlength="200"></el-input>
            <el-date-picker
                v-model="scope.row.rangeDate"
                type="daterange"
                start-placeholder="开始日期"
                end-placeholder="结束日期"
                @change="changeCreateTime(scope.$index)"
                v-if="typeObj[scope.row.parameterName] == 'TIMESTAMP' || typeObj[scope.row.parameterName] == 'DATE'"
                value-format="yyyy-MM-dd" size="mini">
            </el-date-picker>
          </template>  
        </el-table-column>   
        <el-table-column label=""  width="180">
          <template slot-scope="scope">
            <el-button size="mini" class="plusBtn" @click="addArr"><i class="plusBg"></i></el-button>
            <el-button v-if="list.length > 1" size="mini" @click="removeArr(scope.$index)"  class="minusBtn"><i class="minusBg"></i></el-button>
          </template>
        </el-table-column>
      </el-table>

    </div>
    <div class="btn-group">
      <el-button type="primary" size="mini"  @click="save">保存</el-button>
      <el-button size="mini" @click="back">返回</el-button>
    </div>
  </div>
</template>

<script>
  import {apiLogDetail, authorityParamList, authorityParam, saveAuthority} from '@/api/system/index';
  export default {
    name: 'apiAuthority',
    components: {
    },
    props: {

    },
    created() {
    },
    mounted() {
      if (!this.$route.params.id) {
        this.dataId = sessionStorage.getItem('apiAuthorityId')
        this.getDetail()
      }
      
    },
    activated() {
      if (this.$route.params.id) {
        this.dataId = this.$route.params.id
        sessionStorage.setItem('apiAuthorityId', this.dataId)
        this.getDetail()
      }
    },
    watch:{
    },
    data() {
      return {
        dataId: '',
        data: {
          permissionType: '1'//默认开通权限
        },
        total: 0,
        list: [{}],
        endTime:'',
        createTime: [],
        selection: [],
        paramArray: [
        ],
        descObj: {},
        typeObj: {},
        idObj: {}
      }
    },
    computed: {
      elements() {
        return this.$store.state.user.elements
      }
    },
    methods: {
      getDetail() {
        apiLogDetail(this.dataId).then(resp => {
          this.data = resp.data

          //获取参数名称枚举列表
          authorityParam({apiId: resp.data.apiId}).then(res => {
            this.paramArray = res.data.map(v => {
              return {
                value: v.name,
                label: v.name,
                type: v.type
              }
            })
            res.data.forEach(val => {
              this.descObj[val.name] = val.description
            })
            res.data.forEach(val => {
              this.typeObj[val.name] = val.type
            })
            res.data.forEach(val => {
              this.idObj[val.name] = val.id
            })
          })
          //获取权限设置列表
          authorityParamList({apiAuthorizeCode: resp.data.apiAuthorizeCode}).then(res => {
            this.list = res.data.map(v => {
              return {
                parameterName: v.parameterName,
                parameterValue: v.parameterValue,
                requestParametersId: v.requestParametersId,
                parameterType: v.parameterType,
                rangeDate: (v.parameterType == 'DATE' || v.parameterType == 'TIMESTAMP') ? v.parameterValue.split('~') : []
              }
            })
            if (this.list.length == 0) {
              this.list = [{}]
            }
          })
        })
      },
      addArr() {
        let param = {
          parameterName: '',
          parameterValue: '',
          requestParametersId: '',
          parameterType: ''
        }
        this.list.push(param)
      },
      removeArr(index) {
        this.list.splice(index, 1)
      },
      back() {
        this.$router.push({
          name: 'systemLog'
        })
        this.$store.commit('closeTag', 'apiAuthority')
      },
      save() {
        //校验数据范围详细设置
        let paramList = this.list.filter(v => {
          return (v.parameterName || v.parameterValue)
        })
        paramList = paramList.map(v => {
          return {
            parameterName: v.parameterName,
            parameterType: v.parameterType,
            parameterValue: v.parameterValue,
            requestParametersId: v.requestParametersId
          }
        })
        if (this.checkParam(paramList) == 'same') {
           this.$message({
              type: 'warning',
              message: '参数名称不能重复'
            })
          return false
        }
        if (this.checkParam(paramList) == 'empty') {
          this.$message({
            type: 'warning',
            message: '参数名称和数据范围设置不能为空'
          })
          return false
        }
        let param = {
          id: this.dataId,
          permissionType: this.data.permissionType,
          permissionDtoList: this.data.permissionType == '1' ? paramList : []
        }
        saveAuthority(param).then(resp => {
          if (resp.code == 200) {
            this.$message({
              type: 'success',
              message: '保存成功'
            })
            this.$router.push({
              name: 'systemLog'
            })
            this.$store.commit('closeTag', 'apiAuthority')
          } else {
            this.$message({
              type: 'error',
              message: resp.message
            })
          }
        })
      },
      checkParam(data) {
        let result = ''
        let keyArr = []
        data.forEach(v => {
          //非空校验
          if (!v.parameterName || !v.parameterValue) {
            result = 'empty'
          }
          //重复校验
          if (keyArr.indexOf(v.parameterName) > -1) {
            result = 'same'
          } else {
            keyArr.push(v.parameterName)
          }
        })
        return result
      },
      changeCreateTime(index) {
        let rangeDate = this.list[index].rangeDate
        let date = rangeDate[0] + '~' + rangeDate[1]
        this.list[index].parameterValue = date
      },
      changeParam(data, index) {
        this.list[index].requestParametersId = this.idObj[data.parameterName]
        this.list[index].parameterType = this.typeObj[data.parameterName]
      }
   
    }
  }
</script>

<style scoped>

  .btn-box { 
    margin-bottom: 20px;
  }
  .btn-space{
    margin: 0 3px;
  }
</style>


