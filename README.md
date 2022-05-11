# 进程管理——电梯调度系统

[TOC]



### 开发环境

开发语言：Javascript+html+css

开发框架：Vue.js 3.0+Element-plus

开发工具：Vue-cli、Vue-devtools、VScode、Edge

### 实现方法

引入Element-plus组件作为UI，采用Vue3框架进行组件化开发

虽然 Javascript是单线程语言，但通过调用window对象的异步方法 setTimeout 以及 setInterval，使宿主环境（此处为web browser）在JS主线程之外开启多个子线程任务队列，从而完成多线程任务

### 提交文件说明

**dist.zip**: 项目构建后的发布版本

**elevator.zip**: 源代码，其中

+ ./src/App.vue
+ ./src/components/elevator.vue
+ ./src/components/floorboard.vue

   为主要实现代码

### 项目浏览

+ 联网进入 [elevator (pikachudy.github.io)](https://pikachudy.github.io/)在线浏览

+ 解压**dist**文件，在根目录下找到 **index.html** ,使用浏览器打开即可本地浏览

+ 解压**elevator**文件在根目录打开终端

  > 环境配置：
  >
  > 若未安装Node.js，需要先安装Node.js [下载 | Node.js (nodejs.org)](https://nodejs.org/zh-cn/download/)
  >
  > 若未安装Vue cli，在终端输入 `npm install -g @vue/cli` 

  运行 `npm install` 安装项目依赖包

  运行`npm run build` 构建发布版本

  在根目录下会出现**dist**文件夹，使用浏览器打开其中的 **index.html** 即可浏览

### 界面展示&UI说明

#### 电梯内部面板UI说明：

+ 楼层序号两侧感叹号标识并不是显示错误，而会在按下警铃按钮后变为红色以表示发出紧急信号
+ 电梯停靠时，在电梯内部按下当前楼层按钮，电梯会开门（相当于按下了开门键）

+ 电梯内部楼层按钮按下后即被禁用，直到电梯到达该层并停靠、开门后才恢复交互性
+ 电梯在某层停靠时，会自动开门，开门期间电梯不会移动，期间若不按下开关门按钮，则将在五秒后关门，继续运行。右下方蓝色按钮表示电梯门当前状态，并不能点击
+ 电梯运行时开关门按钮将被禁用，无法按下。只有电梯在某一层停靠时才可以按下开关门按钮，以此实现提早关门等操作
+ 电梯显示屏中楼层序号变为红色时说明电梯正在此楼停靠、门处于开启状态；上下箭头变为红色则指示这当前/前一步运行方向

#### 外部呼叫面板UI说明

+ 按钮按下时变为红色并禁用，直到有对应方向的电梯在该楼层停靠时（即调度任务完成）恢复
+ 1楼的下行按钮和20楼的上行按钮只会将电梯呼叫到该层，与1楼上行、20楼下行按钮功能无区别，不会引发bug，此处留下仅为布局美观

### 组件划分

#### 组件预览

| 组件名称   | 负责内容                                                | 子组件               |
| ---------- | ------------------------------------------------------- | -------------------- |
| App        | 接收外部按钮信息、完成外部调度逻辑,协调电梯整体调度逻辑 | Elevator、Floorboard |
| Elevator   | 实现单部电梯系统运行功能及内部调度                      | null                 |
| FloorBoard | 实现电梯外部按钮逻辑                                    | null                 |

#### 各组件变量及功能

##### Elevator

###### 维护变量

```javascript
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
  }
```

###### 基本介绍

组件内部维护了单部电梯运行所需要的信息，每个电梯都有自己的任务队列 ，单部电梯按照队列内容，依据调度算法来运行，减少各部分之间的耦合，尽可能达到高内聚

**mission_floor** 存储电梯的任务序列，电梯依据一定的调度算法，按照队列中内容来运行。
在电梯内部按下按钮则对应楼层将直接加入该队列；在外部按下按钮则通过外部App组件来选择将该楼层的任务放入哪一个电梯的任务队列

在**App.vue**模板中挂载五次，相当于实例化五个相同但互不干扰，独立运行的对象。采用**v-for**循环渲染，给每一个实例化组件赋予**ref**属性以方便父组件区分调用

```javascript
	  <td v-for="i in this.elevator_number" :key="i">
        <elevator ref="elevator_group"
          @upArrive="uparriveHandler"
          @downArrive="downarriveHandler"></elevator>
      </td>
```

##### FloorBoard

###### 维护变量

```js
data() {
    return {
      buttons_upordown: [], // 下标为楼层减1
    };
}
```

###### 简要介绍

**buttons_upordown** 存储楼层按钮对象，对应对象中两按钮按下状态

在按钮可用时按下按钮，会发送事件并传递楼层、方向信息。该事件会触发父组件 **App** 中的调度方法来将该任务加入至合适的电梯的任务序列中（外部调度）

##### App

###### 维护变量

```js
data() {
    return {
      elevator_number: 5, //电梯数目（初始化为5）
      waiting_queue:[]//等待队列，二维数组[0为上行，1为下行][floor-1]，内部存有相应上、下行楼层等待队列计时器返回的对象
    };
  },
```

###### 简要介绍

外部调度任务有时需要等待，此时需要计时器来帮助决定何时放入合适的电梯任务序列中，需要使用 **waiting_queue** 存储相应任务的计时器对象

### 算法设计及实现

#### 电梯内部调度算法设计

电梯的初始方向与第一个加入任务队列的楼层与电梯当前所在楼层的关系有关，此后的调度算法默认保持原来运行方向，直到对应方向上无楼层在目标队列中时再静止或改变方向。

以电梯上行状态举例，电梯没经过一个楼层将会判断该楼层是否处于任务队列中，若在则将在任务队列中将当前楼层删去，进行停靠、开门动作，并在关门后（不能立马检测是因为在停靠期间乘客还会在电梯内部按下自己想去的楼层导致任务序列更新）对任务序列进行检测以决定下一步行驶方向：

1. 若任务队列中目标楼层最大值大于当前楼层，则继续保持上行状态
2. 若任务队列中目标楼层最大值小于于当前楼层，则转换为下行状态
3. 若任务队列为空，则电梯静止，等待新任务加入任务队列

#### 电梯内部调度算法实现

采用异步方法 **setInterval** （在钩子函数中设置及卸载）每隔两秒执行相应回调函数**UpdateStatus**，若此时电梯门为开启状态或任务序列为空则直接跳出，否则根据当前电梯方向更新一次电梯楼层，并检测当前楼层是否在任务序列中

若在则调用方法 **DoorOpen** 将门打开（在 **DoorOpen** 函数中进行判断避免重复开门，主要是为了避免接下来定时器的重复设定），若正常则调用异步方法 **setTimeout** 使5秒后执行 **DoorClose** （在其中添加判断避免重复执行），若正常则关门，并调用方法 **DirectionNext** 确定下一步行进方向，以此来模拟电梯行进及方向选取

```javascript
 methods: {
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

```

#### 电梯外部调度算法设计

外部调度任务主要是将外部呼叫请求根据楼层、方向及之后一段时间内各个电梯状态，将请求放入合适的电梯的任务序列中，使等待时间尽可能缩小

当有新的外部呼叫时，检测所有电梯状态并按以下顺序依次进行判断：

1. 若有电梯在此层停靠且下一步行进方向与呼叫方向相同（或静止），则执行开门操作，任务完成；若有电梯在此层但下一步行进方向与呼叫方向相反则先略过

2. 若无电梯在此层，则检测是否有在当前方向下会经过该层或静止的电梯，若有则将其加入候选队列

   > 假如在8层有上行呼叫请求，则将当前低于8层的处于上行状态的电梯或者处于静止状态的加入候选队列

3. 若候选队列仍为空，则使该呼叫请求等待。并调用异步方法**setInterval**。每隔0.5s检测一次电梯群状态（即进行一次1-2判断）。若有电梯满足要求则加入候选队列，并调用**clearInterval**停止计时器检测

在候选队列中选择与呼叫楼层距离最小的电梯，将呼叫楼层加入其任务队列

采用此方法可以保证选择的电梯到达呼叫层时下一步方向与呼叫方向相同或下一步方向为静止。且在无符合要求的情况下将请求挂起等待，直至满足要求的电梯出现再从中择优，而不是盲目的加入某个电梯的任务队列，提高了效率

且由于电梯自身在当前方向上无任务时，会改变方向或转为静止态（电梯内部调度算法决定的），因此并不会存在所有电梯一直都不满足、候选队列一直为空的情况，也就是说并不会出现某一外部呼叫请求无限等待的情况

#### 电梯外部调度算法实现

```js
methods: {
    
    /*只展示部分方法*/
    
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
      var id=setInterval(this.upWaiting,100,{direction_calling,floor_calling,id});
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
    }
    ...
}
```

在外部面板按钮被按下后，**FloorBoard** 会发出相应事件，被父组件**App**捕获到后调用方法 **upHandler** 实行调度算法，在需要时采用异步方法**setInterval**，每隔0.1s调用一次**upWaiting**方法进行检测，直至某次检测候选队列不为空再调用**clearInterval**清除定时器，完成择优，并将呼叫请求楼层放入相应电梯任务序列

### 算法分析

#### 优点

1. 电梯在当前方向**无任务后便会改变方向或停止**，而不会到达顶层或底层再改变状态，提高效率

2. 在有外部呼叫请求时，**优先将与呼叫方向同向且未经过呼叫楼层的电梯及静止状态的电梯加入候选队列**，再根据距离择优选取，在保证所分配的电梯到达呼叫楼层时，下一步行进方向与呼叫方向同向或静止的前提下能尽快到达

3. 在当前所有电梯都不能满足与呼叫方向同向且未经过呼叫楼层的电梯或为静止状态时**先使呼叫请求等待**，**待出现满足条件的电梯时再加入候选队列**，择优选取。且由于优点一，呼叫请求等待的时间往往不会很长

#### 不足之处

1. 外部请求等待时，一旦碰到方向符合要求的电梯便将其加入候选队列。有些情况下这样的任务分配依旧过早，考虑下面的情况：

   > 假设有两部电梯，第一部目前在15楼，要上行至17楼，随后队列为空；第二部目前在5楼，要上行至8楼，随后队列为空。不考虑电梯停靠时间，每过3s电梯行进一层，现在在3楼外部发出上行呼叫。
   >
   > 按照本项目外部调度算法，一开始两部电梯都不满足，请求等待；
   >
   > 6s后，第一部电梯上行至17楼后静止，此时第二部在7楼继续上行，依据算法则将3楼上行请求加入第一部电梯任务队列，其再经过42s到达第三层，共用时48s；
   >
   > 若分配给第二部电梯，则只需24s

   但实际条件下由于乘客会随时在电梯内选择新楼层，这种情况并不会经常出现

   >例如若在第二部电梯在5-8楼上行期间，电梯内部乘客按下了17楼按钮，那么第二部电梯应当在8楼停靠后继续上行至17层，此时显然没有第一部电梯快

   考虑到这种情况，最终仍采取了当前的外部调度的算法，至少在保证了正确呼叫的前提下尽可能的根据电梯的最新状态进行调度

2. 此处的电梯仍为理想化模型，在调度时未考虑电梯载重量与当前乘客重量；同时也未考虑到电梯停靠时间所带来的影响，下一步完善的话会考虑根据电梯现有的任务队列计算停靠次数，与电梯至目标楼层的距离分别加权求和，择优调度
