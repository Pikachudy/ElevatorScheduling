<template>
  <div class="Border">
    <el-card>
      <el-card class="Display">
        <el-icon :size="20" :class="{ DangerAlert: this.danger }"
          ><warning-filled
        /></el-icon>
        <el-icon :size="20" :class="{ active: this.cur_direction == 1 }"
          ><top
        /></el-icon>
        <span :class="{ active: this.cur_door }">{{ this.cur_floor }}</span>
        <el-icon :size="20" :class="{ active: this.cur_direction == -1 }"
          ><bottom
        /></el-icon>
        <el-icon :size="20" :class="{ DangerAlert: this.danger }"
          ><warning-filled
        /></el-icon>
      </el-card>
      <div class="ButtonGroup">
        <span
          v-for="(button, i) in this.buttons_floor"
          :key="i"
          class="ButtonBox"
        >
          <el-button
            type="info"
            class="MineButton"
            @click="FloorClick(i + 1)"
            :disabled="this.buttons_floor[i]"
            plain
          >
            {{ i + 1 }}
          </el-button>
        </span>
      </div>
      <div class="ButtonGroup">
        <span class="ButtonBox">
          <el-button
            type="warning"
            class="MineButton"
            :disabled="this.cur_moving"
            @click="this.DoorOpen(5000)"
            plain
          >
            <el-icon><caret-left /></el-icon>
            <el-icon><caret-right /></el-icon>
          </el-button>
        </span>
        <span class="ButtonBox">
          <el-button
            type="warning"
            class="MineButton"
            :disabled="this.cur_moving"
            @click="this.DoorClose"
            plain
          >
            <el-icon><caret-right /></el-icon>
            <el-icon><caret-left /></el-icon>
          </el-button>
        </span>
        <span class="ButtonBox">
          <el-button
            type="danger"
            class="MineButton"
            @click="this.DangerAlert"
            plain
          >
            <el-icon><bell-filled /></el-icon>
          </el-button>
        </span>
        <span class="ButtonBox">
          <el-button type="primary" class="MineButton" plain :disabled="true">
            <b>{{ DoorStatus }}</b>
          </el-button>
        </span>
      </div>
    </el-card>
  </div>
</template>

<script>
export default {
  name: "elevator",
  data() {
    return {
      cur_floor: 1, //当前所在楼层
      cur_direction: 0, //当前电梯趋势 1为上行、-1为下行、0为静止
      cur_moving: false, //电梯默认为静止状态、主要为了控制开关门
      cur_door: false, //当前门状态，true为开门
      buttons_floor: [], //长度为20，下标+1表示楼层，内容true为被按下，在EleInit函数初始化
      mission_floor: [], //任务序列，若内部调用无论如何都得接受
      danger: false, //是否报警
      mine_clock: null, //存放setInterval返回的对象便于结束
    };
  },
  methods: {
    EleInit() {
      //初始化按钮群
      for (let i = 0; i < 20; ++i) {
        this.buttons_floor.push(false);
      }
    },

    //选择方向
    DirectionNext() {
      if (!this.mission_floor.length) {
        this.cur_moving = false;
        this.cur_direction = 0;
        return;
      }
      var max = Math.max.apply(Math, this.mission_floor);
      var min = Math.min.apply(Math, this.mission_floor);
      if (this.cur_direction == 1) {
        //电梯上行则看最大值，若做差大于0则继续上行——做差不可能等于0
        if (max - this.cur_floor > 0) {
          return;
        } else {
          this.cur_direction = -1;
        }
      } else if (this.cur_direction == -1) {
        if (min - this.cur_floor < 0) {
          return;
        } else {
          this.cur_direction = 1;
        }
      }
    },

    //开门
    DoorOpen(time) {
      if (this.cur_moving) {
        console.log("有bug！运行的时候怎么能开门呢？");
        return;
      }
      if (this.cur_door) {
        return; //防止重复设定定时器
      }
      this.cur_door = true;
      //两秒后关门
      setTimeout(() => {
        this.DoorClose();
      }, time);
    },

    //关门
    DoorClose() {
      this.DirectionNext();//选取下一个方向
      
      if (this.cur_moving) {
        console.log("有bug！运行的时候怎么能关门呢？");
        return;
      }
      if (!this.cur_door) {
        return; //防止重复
      }
      this.cur_door = false;
      if (this.mission_floor.length == 0) {
        this.cur_moving = false;
      } else {
        this.cur_moving = true;
      }
    },

    //当电梯内有人按下楼层时调用
    FloorClick(i) {
      console.log(i + "被按下！");
      //在当前楼层停止时按下当前楼层按钮会开门
      if (i == this.cur_floor && this.cur_moving == false) {
        console.log("已在当前楼层停止");
        this.DoorOpen(5000); //5秒后关门
        return;
      }
      //判断是否在任务队列中
      if (this.mission_floor.indexOf(i) != -1) {
        console.log("当前楼层已经在任务队列中了！");
        return;
      }
      this.mission_floor.push(i);
      this.buttons_floor[i - 1] = true; //表示为按钮已按下
    },

    //间隔1.5秒执行一次，以此来模拟电梯上下行
    UpdateStatus() {
      console.log("更新电梯状态");
      if (this.cur_door) {
        //若门在开启状态
        return;
      }
      //若任务队列为空，则置为静止状态
      if (this.mission_floor.length == 0) {
        if (this.cur_direction != 0) {
          this.cur_direction = 0;
        }
        if (this.cur_moving != false) {
          this.cur_moving = false;
        }
        return;
      } else {
        this.cur_moving = true;
      }
      this.cur_floor = this.cur_floor + this.cur_direction;
      //若任务队列不空
      //判断当前楼层是否在队列中——即是否到达目的地
      var ArriveCheck = this.mission_floor.indexOf(this.cur_floor);
      //若不在队列中——未到达目的地
      if (ArriveCheck == -1) {
        //若当前为静止状态
        if (this.cur_direction == 0) {
          let UorD = this.mission_floor[0] - this.cur_floor; //取第一个任务与当前楼层做差,不可能为0
          if (UorD > 0) {
            this.cur_direction = 1;
          } else {
            this.cur_direction = -1;
          }
        }
      }
      //若当前不为静止状态——说明此时方向一定正确且未到达
      //若在队列中——已经到达目的地
      else {
        this.mission_floor.splice(ArriveCheck, 1); //在此将楼层从任务队列中取出来
        this.buttons_floor[this.cur_floor - 1] = false;
        this.cur_moving = false; //先停下来才能开门
        
        if (!this.mission_floor.length) {
          //若任务队列已空则静止
          this.cur_moving = false;
          this.cur_direction = 0;
        }

        if(this.cur_direction==0){
          this.$emit("upArrive",this.cur_floor);
          this.$emit("downArrive",this.cur_floor);
        }
        else if(this.cur_direction==1){
          //之前方向为向上
          this.$emit("upArrive",this.cur_floor)
        }
        else if(this.cur_direction==-1){
          //之前方向为向下
          this.$emit("downArrive",this.cur_floor)
        }

        this.DoorOpen(5000);
        //开门5s后关门，选择下一个任务
        
      }
    },

    DangerAlert() {
      this.danger = true;
      setTimeout(() => {
        this.danger = false;
      }, 1000);
    },
  },

  computed: {
    DoorStatus() {
      if (this.cur_door) {
        return "开";
      } else {
        return "关";
      }
    },
  },
  created() {
    this.EleInit();
    this.mine_clock = setInterval(this.UpdateStatus, 1500);
  },
  destroyed() {
    clearInterval(this.mine_clock);
  },
};
</script>

<style scoped>
.Border {
  height: 640px;
  width: 245px;
  display: flex;
  margin-right: 10px;
}
.Display {
  font-size: 150%;
  width: 200px;
  margin-bottom: 20px;
  background-color: azure;
}
.active {
  color: crimson;
}
.ButtonGroup {
  display: flex;
  flex-wrap: wrap;
  width: 200px;
  overflow: hidden;
}
.ButtonBox {
  justify-content: center;
  display: flex;
  width: 35px;
  margin-left: 33px;
  margin-right: 30px;
  margin-bottom: 10px;
}
.MineButton {
  width: 30px;
}
.DangerAlert {
  color: red;
}
</style>
