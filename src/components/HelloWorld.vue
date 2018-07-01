<template>
  <div class="hello">
    <progressBar :isOk=isOK></progressBar>
    <div class="option">
      <el-button size="small" type="primary" @click="reset">重置</el-button>
      <label>时长：</label>
      <el-input :value="timeTxt" readonly="readonly" style="width: 10%;margin-top: 20px"></el-input>
    </div>
    <div class="test">
      <table class="game" @click="handleClick">
        <tr :key="row" v-for="(cols, row) in cellData">
          <cell :key="col" v-for="(cell, col) in cols" :isSelected="cell.isSelected" :isLine="cell.isLine"
                :lineClass="cell.lineClass" :isBlank="cell.isBlank" :className="cell.className"></cell>
        </tr>
      </table>
    </div>
  </div>
</template>

<script>
  import progressBar from '../components/progressbar'
  import Utils from '../common/util'
  import Cell from '../components/cell'
  import config from '../assets/config'

  export default {
    name: 'HelloWorld',
    data() {
      return {
        isOK: false,
        cellData: [],
        currentSelect: null,
        time: null,
        n: 0,
        timeTxt: '00:00:00'
      }
    },
    mounted() {
      this.init()
      this.down()
    },
    methods: {
      down() {
        this.isOK = true
      },
      init() {
        // this.cellData = this.initData()
        this.cellData = this.initData()
      },
      reset() {
        this.init()
        this.resetTime()
      },
      initData() {
        let arr = config.arr
        let cellGroup = arr.map(e => {
          return {
            isBlank: false, // 是否空白方块
            className: e, // 方块的className
            lineClass: '', // 连接线的className
            isLine: false, // 是否显示连接线
            isSelected: false
          }
        })

        // 空白方块
        let blankCell = {
          isBlank: true,
          className: '',
          lineClass: '',
          isLine: false,
          isSelected: false
        }
        // 先根据配置中的方块个数从方块数组中随机取出几条
        let randomCellGroup = Utils.arrayRandom(cellGroup, config.level)

        // 再根据配置中的行和列随机填充一个地图数据
        let cellData = Utils.arrayFillByRandomGroup(config.row * config.col, randomCellGroup)

        // 将数据根据行的大小转为二维数组，然后外部包裹一层空白节点
        let result = Utils.dyadicArrayWrap(Utils.arrayToDyadic(cellData, config.row), blankCell)

        // 最后把行和列的坐标设置到节点上
        result.forEach((cols, row) => {
          cols.forEach((cell, col) => {
            cell.row = row
            cell.col = col
          })
        })
        return result
      },
      handleClick(ev) {
        this.startTime()
        if (ev.target.nodeName !== 'TD') {
          return
        }
        let col = ev.target.cellIndex
        let row = ev.target.parentNode.rowIndex
        // let currentCell = this.cellData[row][col]
        let currentCell = this.cellData[row][col]

        if (currentCell.isBlank === true) return
        this.selectCell(currentCell)
      },
      selectCell(currCell) {
        if (!this.currentSelect) {
          currCell.isSelected = true
          this.currentSelect = currCell
          return
        }
        if (this.currentSelect === currCell) {
          currCell.isSelected = false
          this.currentSelect = null
          return
        }
        let prevCell = this.currentSelect
        if (prevCell.className !== currCell.className) {
          prevCell.isSelected = false
          currCell.isSelected = true
          this.currentSelect = currCell
          return
        }
        let result = this.getLine(prevCell, currCell)
        console.log(result)
        if (result.length === 0) {
          prevCell.isSelected = false
          currCell.isSelected = true
          this.currentSelect = currCell
        } else {
          prevCell.isBlank = true
          currCell.isBlank = true
          prevCell.className = ''
          currCell.className = ''
          prevCell.isSelected = false
          currCell.isSelected = false
          this.currentSelect = null

          this.drawLine(result)
          if(this.checkFinish()){
            this.endTime()
            this.$alert('可以的')
          }
        }
      },
      drawLine(line) {
        line.forEach((e, i) => {
          e.isLine = true
          e.lineClass = this.addLineClass(line[i - 1], e, line[i + 1])
        })
        setTimeout(() => {
          this.hideLine(line)
        }, config.delayTime)
      },
      hideLine(line) {
        line.forEach(e => {
          e.isLine = false
          e.lineClass = ''
        })
      },
      addLineClass(prev, curr, next) {
        let result
        if (!prev) {
          result = 'line-start line-' + this.getDirection(curr, next)
        } else if (!next) {
          result = 'line-end line-' + this.getDirection(curr, prev)
        } else {
          result = 'line-' + this.getDirection(curr, prev) + ' line-' + this.getDirection(curr, next)
        }
        return 'line ' + result
      },
      getDirection(curr, next) {
        return curr.row === next.row ? (curr.col > next.col ? 'l' : 'r') : (curr.row > next.row ? 't' : 'b')
      },
      // 检查两个点是否连通的
      checkBeeline(start, end) {
        let result = true
        // 先获取这两个点的连线
        let beeline = this.getBeeline(start, end)
        for (let lineCell of beeline) {
          // 然后检查线上是否存在非空节点，如果存在非空节点说明这两点之间是无法连通的
          if (!lineCell.isBlank) {
            result = false
            break
          }
        }
        return result
      },
      // 获得从 start 到 end 的直线路径数组

      // 查找，通过一个数组与需要要查找的数组下标来计算
      checkCell(arr, index) {
        let set = new Set()
        // 向后查找
        for (let i = index - 1; i >= 0; i--) {
          let cell = arr[i]
          // 判断className相同或者是空白方块
          if (cell.className === arr[index].className || cell.isBlank) {
            set.add(cell)
          }
          // 如果不是空白方块那么终止查找
          if (!cell.isBlank) {
            break
          }
        }
        // 向前查找
        for (let i = index + 1, l = arr.length; i < l; i++) {
          let cell = arr[i]
          if (cell.className === arr[index].className || cell.isBlank) {
            set.add(cell)
          }
          if (!cell.isBlank) {
            break
          }
        }
        return set
      },

      getHorizontalLine(curr) {
        return this.checkCell(this.cellData[curr.row], curr.col)
      },
      getVerticalLine(curr) {
        return this.checkCell(this.cellData.map(e => e[curr.col]), curr.row)
      },
      getBeeline(start, end) {
        let startIndex
        let endIndex
        let arr
        if (start.row === end.row) {
          startIndex = start.col
          endIndex = end.col
          arr = this.cellData[start.row]
        } else {
          startIndex = start.row
          endIndex = end.row
          arr = this.cellData.map(e => e[start.col])
        }
        return startIndex < endIndex ? arr.slice(startIndex, endIndex + 1) : arr.slice(endIndex, startIndex + 1).reverse()
      },

      getLine(prev, curr) {
        let result = []
        let prevH = this.getHorizontalLine(prev)
        if (prevH.has(curr)) {
          return this.getBeeline(prev, curr)
        }
        let prevV = this.getVerticalLine(prev)
        if (prevV.has(curr)) {
          return this.getBeeline(prev, curr)
        }
        let currH = this.getHorizontalLine(curr)
        let currV = this.getVerticalLine(curr)
        if ((!prevH.size && !prevV.size) || (!currH.size && !currV.size)) return result

        let intersection = this.getIntersection(prevH, currV) || this.getIntersection(prevV, currH)
        if (intersection) {
          return this.getBeeline(prev, intersection).concat(this.getBeeline(intersection, curr).slice(1))
        }
        let intersectionArr = this.getIntersectionArr(prevH, currH, prev.row, curr.row, true)
        if (intersectionArr.length === 0) {
          intersectionArr = this.getIntersectionArr(prevV, currV, prev.col, curr.col, false)
        }
        if (intersectionArr.length > 0) {
          result = this.getBeeline(prev, intersectionArr[0]).concat(this.getBeeline(intersectionArr[0], intersectionArr[1]).slice(1)).concat(this.getBeeline(intersectionArr[1], curr).slice(1))
        }
        return result
      },

      getIntersection(prevLine, currLine) {
        let intersection = null
        for (let cell of prevLine) {
          if (currLine.has(cell) && cell.isBlank) {
            intersection = cell
            break
          }
        }
        return intersection
      },
      getIntersectionArr(prevLine, currLine, prevIndex, currIndex, isRow) {
        let result = []
        if (!prevLine.size || !currLine.size) {
          return result
        }
        let keyCode = isRow ? 'col' : 'row'
        let prevFullLine = isRow ? this.cellData[prevIndex] : this.cellData.map(e => e[prevIndex])
        let currFullLine = isRow ? this.cellData[currIndex] : this.cellData.map(e => e[currIndex])
        for (let prevCell of prevLine) {
          if (!prevCell.isBlank) continue

          let target = currFullLine[prevCell[keyCode]]
          if (currLine.has(target)) {
            let index = target[keyCode]
            let isBeeLine = this.checkBeeline(prevFullLine[index], currFullLine[index])
            if (isBeeLine) {
              result = [prevFullLine[index], currFullLine[index]]
              break
            }
          }
        }
        return result
      },
      startTime() {
        clearInterval(this.time)
        this.time = setInterval(() => {
          this.n++
          let m = parseInt(this.n / 3600)
          let s = parseInt(this.n / 60 % 60)
          let M = parseInt(this.n % 60)
          this.timeTxt = `${this.toDub(m)}:${this.toDub(s)}:${this.toDub(M)}`
        }, 1000 / 60)
      },
      endTime() {
        clearInterval(this.time)
      },
      resetTime() {
        clearInterval(this.time)
        this.n = 0
        this.timeTxt = '00:00:00'
      },
      toDub(n) {
        return n < 10 ? "0" + n : "" + n
      },
      checkFinish(){
        let isFinish=true
        for(let i=0;i<config.row+1;i++){
          for(let j=0;j<config.col+1;j++){
            let blank=this.cellData[i][j].isBlank
            if(!blank) return false
          }
        }
        return isFinish
      }

    },
    components: {
      progressBar,
      Cell
    }
  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>
  .game {
    margin: 0 auto;

    border-collapse: separate;
    user-select: none;
    text-align: center;
    /*margin-top: 65px;*/
    border-spacing: 2px;
  }

  .option {

  }

  .game-config {
    position: fixed;
    left: 0;
    right: 0;
    top: 0;
    z-index: 100;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 20px 0;
    background-color: #fff;
    flex-wrap: wrap;
  }

  /* theme test */
  .test {
    font-family: calibri, Microsoft Yahei;
  }

  .test td {
    width: 44px;
    height: 44px;
    border: 2px solid transparent;
    background-color: #0cf;
    cursor: pointer;
    border-radius: 4px;
    font-size: 20px;
    color: #fff;
    position: relative;
  }

  .test .blank {
    background-color: transparent;
    cursor: default;
  }

  .test .selected {
    border: 2px solid #e4a042;
  }

  .test .line {
    position: absolute;
    top: 0;
    right: 0;
    left: 0;
    bottom: 0;
  }

  .test .line:before, .test .line:after {
    content: '';
    display: block;
    position: absolute;
    background-color: #e4a042;
  }

  .test .line.line-l:before, .test .line.line-r:before {
    width: calc(50% + 4px);
    height: 4px;
    top: 50%;
    right: 50%;
    margin-top: -2px;
  }

  .test .line.line-r:before {
    right: -4px;
  }

  .test .line.line-l.line-r:before {
    width: calc(100% + 8px);
    left: -4px;
    right: -4px;
  }

  .test .line.line-t:after, .test .line.line-b:after {
    width: 4px;
    height: calc(50% + 4px);
    left: 50%;
    bottom: 50%;
    margin-left: -2px;
  }

  .test .line.line-b:after {
    bottom: -4px;
  }

  .test .line.line-t.line-b:after {
    height: calc(100% + 8px);
    top: -4px;
    bottom: -4px;
  }

  .test .line.line-t.line-r:before, .test .line.line-b.line-r:before {
    width: calc(50% + 6px);
    border-top-left-radius: 2px;
    border-bottom-left-radius: 2px;
  }

  .test .line.line-t.line-l:before, .test .line.line-b.line-l:before {
    width: calc(50% + 6px);
    margin-right: -2px;
    border-top-right-radius: 2px;
    border-bottom-right-radius: 2px;
  }

  .test .line.line-start.line-l:before, .test .line.line-start.line-r:before, .test .line.line-end.line-l:before, .test .line.line-end.line-r:before {
    width: calc(20% + 4px);
    right: 80%;
  }

  .test .line.line-start.line-r:before, .test .line.line-end.line-r:before {
    left: 80%;
  }

  .test .line.line-start.line-l:after, .test .line.line-start.line-r:after, .test .line.line-end.line-l:after, .test .line.line-end.line-r:after {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    right: 80%;
    top: 50%;
    margin-top: -5px;
    margin-right: -5px;
  }

  .test .line.line-start.line-r:after, .test .line.line-end.line-r:after {
    margin-left: -5px;
    left: 80%;
  }

  .test .line.line-start.line-t:after, .test .line.line-start.line-b:after, .test .line.line-end.line-t:after, .test .line.line-end.line-b:after {
    height: calc(20% + 4px);
    bottom: 80%;
  }

  .test .line.line-start.line-b:after, .test .line.line-end.line-b:after {
    top: 80%;
  }

  .test .line.line-start.line-t:before, .test .line.line-start.line-b:before, .test .line.line-end.line-t:before, .test .line.line-end.line-b:before {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    bottom: 80%;
    left: 50%;
    margin-left: -5px;
    margin-bottom: -5px;
  }

  .test .line.line-start.line-b:before, .test .line.line-end.line-b:before {
    margin-top: -5px;
    top: 80%;
  }
  .test .a{
    background:url("../assets/logo.png")center no-repeat;
    background-size:100% 100%
  }
</style>
