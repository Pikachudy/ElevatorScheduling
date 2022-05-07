<template>
  <div class="Border">
    <el-card>
      <div class="ButtonGroup">
        <span
          v-for="(floor_quest, i) in this.buttons_upordown"
          :key="i"
          class="ButtonBox"
        >
          <el-button-group class="ml-4">
            <el-button
              type="primary"
              :class="{ MineButton: true, active: floor_quest.up }"
              @click="this.ButtonClick(i+1,1)"
              plain
            >
              <el-icon><arrow-up-bold /></el-icon>
            </el-button>
            <el-button type="primary" class="MineButton" :disabled="true" plain>
              <b>{{ i + 1 }}</b>
            </el-button>
            <el-button
              type="primary"
              :class="{ MineButton: true, active: floor_quest.down }"
              @click="this.ButtonClick(i+1,-1)"
              plain
            >
              <el-icon><arrow-down-bold /></el-icon>
            </el-button>
          </el-button-group>
        </span>
      </div>
    </el-card>
  </div>
</template>

<script>
export default {
  name: "floorboard",
  data() {
    return {
      buttons_upordown: [], // 下标为楼层减1
    };
  },
  methods: {
    BoardInit() {
      for (let i = 0; i < 20; i++) {
        let request = {
          up: false,
          down: false
        };
        this.buttons_upordown.push(request);
      }
    },

    ButtonClick(floor,direction){
        //更改数组中相应对象值，并发出事件交由外部处理
        if(direction==1&&this.buttons_upordown[floor-1].up!=true){
            this.buttons_upordown[floor-1].up=true;
            this.$emit('upRequest',{
                floor_calling : floor,
                direction_calling : direction
            });
        }
        else if(direction==-1&&this.buttons_upordown[floor-1].down!=true){
            this.buttons_upordown[floor-1].down=true;
            this.$emit('downRequest',{
                floor_calling : floor,
                direction_calling : direction
            });
        };
    }

  },
  created() {
    this.BoardInit();
  },
};
</script>

<style scoped>
.Border {
  height: 640px;
  width: 245px;
  display: flex;
  margin-right: 10px;
  margin-top: 0px;
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
  height: 33px;
  margin-left: 3px;
  margin-right: 1px;
  margin-bottom: 30px;
}
.MineButton {
  width: 32px;
}
</style>
