<template>
  <!-- <img alt="Vue logo" src="./assets/logo.png">
  <HelloWorld msg="Welcome to Your Vue.js App"/> -->
  <div>
    <tr>
      <td v-for="i in this.elevator_number" :key="i">
        <elevator ref="elevator_group"></elevator>
      </td>
      <td valign="top">
        <floorboard
          ref="floor_board"
          @upRequest="upHandler"
          @downRequest="downHandler"
        ></floorboard>
      </td>
    </tr>
  </div>
</template>

<script>
import elevator from "./components/elevator.vue";
import floorboard from "./components/floorboard.vue";
export default {
  name: "App",
  components: {
    elevator,
    floorboard,
  },
  data() {
    return {
      elevator_number: 5, //五部电梯
    };
  },
  methods: {
    minDistenceEle(candidateEles,floor_calling,elevators){
      //计算最小距离的电梯
      var minDistence = 100;//一个不可能的极大数
      var minElevator = 0;
      for(let i=0;i<candidateEles.length;i++){
        if(Math.abs(elevators[i].cur_floor-floor_calling)<minDistence){
          minElevator = i;
        }
      }
      return minElevator;
    },

    upHandler(arg) {
      let elevators=this.$refs.elevator_group;
      let {direction_calling,floor_calling}=arg;
      var candidateEles=[];
      //处理外部调度信息
      for(let i=0;i<5;i++){
        if(elevators[i].cur_direction==0||elevators[i].cur_direction==direction_calling){
          //若有静止或同向电梯:
          if(elevators[i].cur_floor==floor_calling&&elevators[i].cur_direction==0){
          //若电梯就在此层则开门就直接ok
            elevators[i].DoorOpen(2000);
            return;
          }
          //若不在此层则加入候选队列
          candidateEles.push(i);
        }
      }
      if(candidateEles.length!=0){
        //若已有则挑选然后加入任务队列
        let elevator_index = this.minDistenceEle(candidateEles,floor_calling,elevators);
        if(this.$refs.elevator_group[elevator_index].mission_floor.indexOf(floor_calling)==-1){
          this.$refs.elevator_group[elevator_index].mission_floor.push(floor_calling);
        }
        return 
      }
      //处理无候选结果
      
    },
    downHandler(arg) {
      //处理外部调度信息
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
.EvelatorBox {
  display: flex;
  height: 800px;
}
.ElevatorCard {
  display: flex;
}
</style>
