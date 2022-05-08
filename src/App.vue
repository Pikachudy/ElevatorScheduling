<template>
  <!-- <img alt="Vue logo" src="./assets/logo.png">
  <HelloWorld msg="Welcome to Your Vue.js App"/> -->
  <div>
    <tr>
      <td v-for="i in this.elevator_number" :key="i">
        <elevator ref="elevator_group"
          @upArrive="uparriveHandler"
          @downArrive="downarriveHandler"></elevator>
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
      waiting_queue:[]//等待队列，二维数组[0为上行，1为下行][floor-1]，内部存有相应上、下行楼层等待队列计时器返回的对象
    };
  },
  created(){
    for(let i=0;i<2;i++){
      let temArray=[];
      for(let j=0;j<20;j++){
        temArray[j]=null;
      }
      this.waiting_queue[i]=temArray;
    }
  },
  mounted(){
    this.$nextTick(()=>{
      for(var i=0;i<5;i++){
        this.$refs.elevator_group[i].cur_floor=1+i*4;
      }
    })
  },
  methods: {
    uparriveHandler(floor){
      this.$refs.floor_board.buttons_upordown[floor-1].up=false;
    },
    downarriveHandler(floor){
      this.$refs.floor_board.buttons_upordown[floor-1].down=false;
    },
    minDistenceEle(candidateEles, floor_calling, elevators) {
      //计算最小距离的电梯
      var minDistence = 100; //一个不可能的极大数
      var minElevator = 0;
      for (let i = 0; i < candidateEles.length; i++) {
        if (Math.abs(elevators[i].cur_floor - floor_calling) < minDistence) {
          minDistence = Math.abs(elevators[i].cur_floor - floor_calling);
          minElevator = i;
        }
      }
      return candidateEles[minElevator];
    },

    upHandler(arg) {
      let elevators = this.$refs.elevator_group;
      let { direction_calling, floor_calling } = arg;
      var candidateEles = [];
      //处理外部调度信息
      for (let i = 0; i < 5; i++) {
        if (elevators[i].cur_direction == 0) {
          //若有静止电梯
          if (elevators[i].cur_floor == floor_calling) {
            //若电梯就静止在此层则开门就直接ok
            this.uparriveHandler(floor_calling);
            elevators[i].DoorOpen(2000);
            return;
          }
          //若静止但不在此层则加入候选队列
          candidateEles.push(i);
        }

        if (
          elevators[i].cur_direction == direction_calling &&
          floor_calling - elevators[i].cur_floor > 0
        ) {
          //若电梯同向且尚未经过此层则加入候选队列
          candidateEles.push(i);
        }
      }

      if (candidateEles.length != 0) {
        //若已有则挑选然后加入任务队列
        let elevator_index = this.minDistenceEle(
          candidateEles,
          floor_calling,
          elevators
        );
        if (
          this.$refs.elevator_group[elevator_index].mission_floor.indexOf(
            floor_calling
          ) == -1
        ) {
          this.$refs.elevator_group[elevator_index].mission_floor.push(
            floor_calling
          );
        }
        return;
      }
      //处理无候选结果,此时情况为电梯反向或虽然同向但已经经过了该层
      //此时丢入等待序列，每0.5秒检测一次电梯状态，直至出现静止电梯或同向未经过该层电梯再结束子线程
      console.log(floor_calling+"层的"+direction_calling+"请求放入等待队列");
      var id=setInterval(this.upWaiting,500,{direction_calling,floor_calling,id});//通过闭包性质将id传入回调函数
      this.waiting_queue[0][floor_calling-1]=id;
    },

    upWaiting(arg) {
      let { direction_calling, floor_calling } = arg;
      console.log("检测"+floor_calling+"层的"+direction_calling+"请求");
      let elevators = this.$refs.elevator_group;
      var candidateEles = [];
      var id =  this.waiting_queue[0][floor_calling-1];
      //处理外部调度信息
      for (let i = 0; i < 5; i++) {
        if (elevators[i].cur_direction == 0) {
          //若有静止电梯
          if (elevators[i].cur_floor == floor_calling) {
            //若电梯就静止在此层则开门就直接ok
            elevators[i].DoorOpen(2000);
            console.log(floor_calling+"层的"+direction_calling+"请求已满足，在"+ i+"号电梯");
            clearInterval(id);    //停用计时器
            return;
          }
          //若静止但不在此层则加入候选队列
          candidateEles.push(i);
        }

        if (
          elevators[i].cur_direction == direction_calling &&
          floor_calling - elevators[i].cur_floor > 0
        ) {
          //若电梯同向且尚未经过此层则加入候选队列
          candidateEles.push(i);
        }
      }

      if (candidateEles.length != 0) {
        //若已有则挑选然后加入任务队列
        let elevator_index = this.minDistenceEle(
          candidateEles,
          floor_calling,
          elevators
        );
        if (
          this.$refs.elevator_group[elevator_index].mission_floor.indexOf(
            floor_calling
          ) == -1
        ) {
          this.$refs.elevator_group[elevator_index].mission_floor.push(
            floor_calling
          );
        }
        clearInterval(id);//停用计时器
        console.log(floor_calling+"层的"+direction_calling+"请求已放入"+ elevator_index+"号电梯任务队列");
        return;
      }
      console.log(floor_calling+"层的"+direction_calling+"请求暂未满足 继续等待");
      return;
    },

    downHandler(arg) {
      //处理外部调度信息
      let elevators = this.$refs.elevator_group;
      let { direction_calling, floor_calling } = arg;
      var candidateEles = [];
      //处理外部调度信息
      for (let i = 0; i < 5; i++) {
        if (elevators[i].cur_direction == 0) {
          //若有静止电梯
          if (elevators[i].cur_floor == floor_calling) {
            //若电梯就静止在此层则开门就直接ok
            this.downarriveHandler(floor_calling);
            elevators[i].DoorOpen(2000);
            return;
          }
          //若静止但不在此层则加入候选队列
          candidateEles.push(i);
        }

        if (
          elevators[i].cur_direction == direction_calling &&
          floor_calling - elevators[i].cur_floor < 0
        ) {
          //若电梯同向且尚未经过此层则加入候选队列
          candidateEles.push(i);
        }
      }

      if (candidateEles.length != 0) {
        //若已有则挑选然后加入任务队列
        let elevator_index = this.minDistenceEle(
          candidateEles,
          floor_calling,
          elevators
        );
        if (
          this.$refs.elevator_group[elevator_index].mission_floor.indexOf(
            floor_calling
          ) == -1
        ) {
          this.$refs.elevator_group[elevator_index].mission_floor.push(
            floor_calling
          );
        }
        return;
      }
      //处理无候选结果,此时情况为电梯反向或虽然同向但已经经过了该层
      //此时丢入等待序列，每0.5秒检测一次电梯状态，直至出现静止电梯或同向未经过该层电梯再结束子线程
      console.log(floor_calling+"层的"+direction_calling+"请求放入等待队列");
      var id=setInterval(this.downWaiting,500,{direction_calling,floor_calling,id});//通过闭包性质将id传入回调函数
      this.waiting_queue[1][floor_calling-1]=id;
    },
    
    downWaiting(arg){
      let { direction_calling, floor_calling } = arg;
      console.log("检测"+floor_calling+"层的"+direction_calling+"请求");
      let elevators = this.$refs.elevator_group;
      var candidateEles = [];
      var id =  this.waiting_queue[1][floor_calling-1];
      //处理外部调度信息
      for (let i = 0; i < 5; i++) {
        if (elevators[i].cur_direction == 0) {
          //若有静止电梯
          if (elevators[i].cur_floor == floor_calling) {
            //若电梯就静止在此层则开门就直接ok
            elevators[i].DoorOpen(2000);
            console.log(floor_calling+"层的"+direction_calling+"请求已满足，在"+ i+"号电梯");
            clearInterval(id);    //停用计时器
            return;
          }
          //若静止但不在此层则加入候选队列
          candidateEles.push(i);
        }

        if (
          elevators[i].cur_direction == direction_calling &&
          floor_calling - elevators[i].cur_floor < 0
        ) {
          //若电梯同向且尚未经过此层则加入候选队列
          candidateEles.push(i);
        }
      }

      if (candidateEles.length != 0) {
        //若已有则挑选然后加入任务队列
        let elevator_index = this.minDistenceEle(
          candidateEles,
          floor_calling,
          elevators
        );
        if (
          this.$refs.elevator_group[elevator_index].mission_floor.indexOf(
            floor_calling
          ) == -1
        ) {
          this.$refs.elevator_group[elevator_index].mission_floor.push(
            floor_calling
          );
        }
        clearInterval(id);//停用计时器
        console.log(floor_calling+"层的"+direction_calling+"请求已放入"+ elevator_index+"号电梯任务队列");
        return;
      }
      console.log(floor_calling+"层的"+direction_calling+"请求暂未满足 继续等待");
      return;
    }
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
