<template>
    <div class="Border">
      <el-card>
        <el-card class="Display">
          <el-icon :size=20 
                   :class="{active: this.cur_direction==1}"><top /></el-icon>
          <span :class="{active: !this.cur_moving}">{{this.cur_floor}}</span>
          <el-icon :size=20
                   :class="{active: this.cur_direction==-1}"><bottom /></el-icon>
        </el-card>
        <div class="ButtonGroup">
          <span v-for="(button,i) in this.buttons_floor"
              :key="i"
              class="ButtonBox">
            <el-button type="info" class="MineButton" @click="FloorClick(i+1)" :disabled="this.buttons_floor[i]" plain> {{i+1}} </el-button>
          </span>
        </div>
        <div class="ButtonGroup">
          <span class="ButtonBox">
            <el-button type="warning" class="MineButton" :disabled="this.cur_moving" plain>
              <el-icon><caret-left /></el-icon>
              <el-icon><caret-right /></el-icon>
            </el-button>
          </span>
          <span class="ButtonBox" >
            <el-button type="warning" class="MineButton" :disabled="this.cur_moving" plain>
              <el-icon><caret-right /></el-icon>
              <el-icon><caret-left /></el-icon>
            </el-button>
          </span>
          <span class="ButtonBox">
            <el-button type="danger" class="MineButton" plain>
              <el-icon><bell-filled /></el-icon>
            </el-button>
          </span>
          <span class="ButtonBox">
            <el-button type="primary" class="MineButton" plain :disabled="true">
              <b>{{ DoorStatus}}</b>
            </el-button>
          </span>
        </div>
      </el-card>
      <div style="display: flex">
        
      </div>
    </div>
</template>

<script>
export default {
    name:'elevator',
    data(){
      return{
        cur_floor : 1,//当前所在楼层
        cur_direction : 0,//当前电梯趋势 1为上行、-1为下行、0为静止
        cur_moving: false,//电梯默认为静止状态、主要为了控制开关门
        cur_door: false,//当前门状态，true为开门
        buttons_floor:[],//长度为20，下标+1表示楼层，内容true为被按下，在EleInit函数初始化
        mission_floor:[] //任务序列，若内部调用无论如何都得接受
      }
    },
    methods:{
      EleInit(){
        //初始化按钮群
        for(let i=0;i<20;++i){
          this.buttons_floor.push(false);
        }
      },
      //当电梯内有人按下楼层时调用
      FloorClick(i){
        console.log(i+"被按下！")
        //判断是否在任务队列中
        if(this.mission_floor.indexOf(i)!=-1){
          console.log("当前楼层已经在任务队列中了！");
          return;
        }
        this.mission_floor.push(i);
        this.buttons_floor[i-1] = true;//表示为按钮已按下
      }
    },
    computed:{
      DoorStatus(){
        if(this.cur_door){
          return "开";
        }
        else{
          return "关"
        }
      }
    },
    created(){
      this.EleInit();
    }
    
}
</script>

<style scoped>
    .Border{
      height: 640px;
      width: 245px;
      display: flex;
    }
    .Display{
      font-size: 150%;
      width: 200px;
      margin-bottom: 20px;
      background-color:azure
    }
    .active{
      color: orange;
    }
    .ButtonGroup{
      display: flex;
      flex-wrap: wrap; 
      width: 200px;
      overflow: hidden;
    }
    .ButtonBox{
     justify-content: center;
     display: flex;
     width: 35px;
     margin-left: 33px;
     margin-right: 30px;
     margin-bottom: 10px;
    }
    .MineButton{
      width: 30px;
    }
</style>