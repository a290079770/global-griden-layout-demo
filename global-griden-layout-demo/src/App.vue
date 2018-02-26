<template>
    <div class="book-detail" :style="{width:winWidth*0.95 +'px',height:winHeight +'px'}" @selectstart.prevent="" @mouseup="endExchange($event)" @mousemove="overExchange($event)">
       <el-button type="primary" @click="addTab('','')">添加固定窗口 +</el-button>
       <el-button type="success" @click="addCoverTab('','')">添加覆盖窗口 +</el-button>
       <el-button @click="dialogTableVisible = true" >历史窗口</el-button>
       <el-button @click="openActiveSearch" >打开当前tab的文内检索</el-button>

       <div class="container" @mouseover="">
          <div 
           class="panel" 
          :style="{width:1/panels.length * winWidth*0.95 + 'px',background:index == activePanelIndex?'#e4ebf5':'white',opacity:index == movePanelIndex?'0':'1'}" 
           v-for="(item,index) in panels"
           :id="'panel'+index" 
           @click="changeActivePanel(index)"
           @mousedown="startExchange($event,index)"
           >
             <div class="tab-list" >
                <ul class="tab-cont" :style="{maxWidth:1/panels.length * winWidth*0.95 - 90 + 'px'}">
                  <li class="tab" 
                      v-for="(tabItem,tabIndex) in item.tabs" 
                      :id="tabItem.id"
                      :style="{background:tabIndex == item.activeTabIndex?'#ccc':'#eee'}"  
                      @click="changeActiveTab(item,index,tabIndex)"
                  >
                      {{tabItem.title}}

                      <span class="close-btn" @click.stop="closeCurTab(item.tabs,tabIndex,index)"> X </span>
                  </li>
                </ul>

                <div class="panel-control">
                    <i class="el-icon-date" @click.stop="splitTabToPanel(index)"></i>

                     <el-dropdown trigger="click" @command="handleClose">
                        <span class="el-dropdown-link">
                           <i class="el-icon-more"></i> 
                        </span>
                        <el-dropdown-menu slot="dropdown">
                          <el-dropdown-item :command="index+'-'+1">关闭所有选项卡</el-dropdown-item>
                          <el-dropdown-item :command="index+'-'+2" :disabled="item.tabs.length == 1">关闭其他选项卡</el-dropdown-item>
                        </el-dropdown-menu>
                     </el-dropdown>
                    
                </div>
             </div>

             <div class="panel-content" >
                <div :id="'panelContent'+index+'-'+item.activeTabIndex">{{item.tabs[item.activeTabIndex].content}}</div>

                <div 
                class="panel-search" 
                :id="'panelSearch'+index+'-'+item.activeTabIndex" 
                v-for="(tabItem,tabIndex) in item.tabs" 
                :data-id="tabItem.id"
                v-if="item.tabs[item.activeTabIndex].id == tabItem.id"
                v-show="tabItem.isShowSearch"
                >
                
                  {{tabItem.content+'的文内检索窗口'}}
                </div>
             </div>
      
          </div>
          

          <!-- 这里预留一个div，用作不同div之间切换使用 -->
          <div 
           class="panel move-panel" 
           v-show="isStartMove"
           :style="{width:1/panels.length * winWidth*0.95 + 'px',top:parentOffsetTop + 'px',left:movePanelLeft +'px'}" 
           >
             <div class="tab-list" >
                <ul class="tab-cont" :style="{maxWidth:100/panels.length + '%'}">
                  <li class="tab" 
                      v-for="(tabItem,tabIndex) in movePanel.tabs" 
                      :style="{background:tabIndex == movePanel.activeTabIndex?'#ccc':'#eee'}"  
                  >
                      {{tabItem.title}}

                      <span class="close-btn"> X </span>
                  </li>
                </ul>

                <div class="panel-control">
                    <i class="el-icon-date"></i>
                    <i class="el-icon-more"></i> 
                </div>
             </div>

             <div class="panel-content" >
                {{movePanel.tabs[movePanel.activeTabIndex].content}}
             </div>
      
          </div>


       </div>


       <!-- 历史记录 -->
       <el-dialog title="历史窗口" :visible.sync="dialogTableVisible">
          <el-table :data="historyPanel" @row-click="restoreHistoryPanel" style="cursor:pointer">
            <el-table-column property="title" label="窗口名" width="150"></el-table-column>
            <el-table-column property="content" label="窗口内容"></el-table-column>
            <el-table-column property="isCover" label="窗口类型">
              <template scope="props">
                  {{props.row.isCover?'覆盖':'固定'}}
              </template>
            </el-table-column>
          </el-table>

          <div slot="footer" class="dialog-footer">
            <el-button @click="dialogTableVisible = false">取 消</el-button>
            <el-button type="primary" @click="dialogTableVisible = false" :disabled="true">确 定</el-button>
          </div>
        </el-dialog>
    </div>
</template>
<script>
import Split from 'split.js';
export default {
    name: 'bookDetail',
    data() {
        return {
          winWidth:window.innerWidth,
          winHeight:window.innerHeight,
          panels:[
             // {
             //  name:activeTab.title,
             //  tabs:[
             //     {
             //      title:activeTab.title,
             //      content:activeTab.content
             //      isCover:false,
             //      id:''
             //     }
             //  ],
             //  activeTabIndex:0,
             // }
          ],

          activePanelIndex:0,
          historyPanel:[
            // {
            //    title:'',
            //    content:'',
            //    isCover:false
            // }
          ],


          dialogTableVisible: false,
          instance:null,
          movePanelIndex:-1,
          startMoveTimer:null,
          movePanel:{
            name:'',
            tabs:[
               {
                title:'',
                content:''
               }
            ],
            activeTabIndex:0
          },
          isStartMove:false,
          parentOffsetLeft:0,
          parentOffsetTop:0,
          movePanelLeft:0
        }
    },
    methods:{
       setWindow() {
          this.winWidth = window.innerWidth;
          this.winHeight = window.innerHeight;
       },
       //改变活跃窗口
       changeActivePanel(index) {
         this.activePanelIndex = index;
       },
       
       //改变活跃tab
       changeActiveTab(item,index,tabindex) {
         item.activeTabIndex = tabindex;
         this.$set(this.panels,index,item);
       },
       //为窗口生成唯一id,32位
       buildIds() {
         const STR = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
         let resStr = '';
         for(let i = 0; i < 32 ; i++) {
             var a = Math.ceil(Math.random()*61);
             resStr += STR[a];
         }

         return resStr;
       },

       //添加tab类型的窗口，在聚焦窗口添加，这种窗口是不覆盖的
       addTab(title ='',content ='',isCover=false) {
         //如果是初始状态
         if(this.panels.length == 0 && !title && !content) {
            //第一次添加窗口
            this.panels.push({
              name:'panel1',
              tabs:[
                 {
                  title:'固定tab1',
                  content:'这是固定tab1',
                  isCover:false,
                  id:this.buildIds(),
                  isShowSearch:false
                 }
              ],
              activeTabIndex:0
             });

            this.historyPanel.unshift({
                  title:'固定tab1',
                  content:'这是固定tab1',
                  isCover:false,
                  isShowSearch:false
                 })
            
            this.activePanelIndex = 0;
            return;
         }else if(this.panels.length == 0 && (title || content)) {
            //历史窗口进入时，若没有窗口
            this.panels.push({
              name:title,
              tabs:[
                 {
                  title:title,
                  content:content,
                  isCover:isCover,
                  id:this.buildIds(),
                  isShowSearch:false
                 }
              ],
              activeTabIndex:0
             });

            this.activePanelIndex = 0;
            return;
         }

         //已存在窗口
         let addPanelTabs = this.panels[this.activePanelIndex].tabs;
         
         let randomIndex = Math.floor(Math.random()*100);

         let newTabTitle = title?title:'固定tab'+ randomIndex;
         let newTabContent = content?content:'这是固定tab' + randomIndex;

         let newTab = {
                        title:newTabTitle,
                        content:newTabContent,
                        isCover:isCover,
                        id:this.buildIds(),
                        isShowSearch:false
                       }

         addPanelTabs.push(newTab);

         this.panels[this.activePanelIndex].activeTabIndex = addPanelTabs.length - 1;

         
         if(!title && !content) {
           this.historyPanel.unshift(newTab);
         }
       },

       //添加覆盖类型的tab窗口
       addCoverTab(title ='',content ='') {
         //如果是初始状态
         if(this.panels.length == 0 && !title && !content) {
            //第一次添加窗口
            this.panels.push({
              name:'panel1',
              tabs:[
                 {
                  title:'覆盖tab1',
                  content:'这是覆盖tab1',
                  isCover:true,
                  id:this.buildIds(),
                  isShowSearch:false
                 }
              ],
              activeTabIndex:0
             });

            this.historyPanel.unshift({
                  title:'覆盖tab1',
                  content:'这是覆盖tab1',
                  isCover:true,
                  isShowSearch:false
                 })
            
            this.activePanelIndex = 0;
            return;
         }else if(this.panels.length == 0 && (title || content)) {
            //历史窗口进入时，若没有窗口
            this.panels.push({
              name:title,
              tabs:[
                 {
                  title:title,
                  content:content,
                  isCover:true,
                  id:this.buildIds(),
                  isShowSearch:false
                 }
              ],
              activeTabIndex:0
             });

            this.activePanelIndex = 0;
            return;
         }

         //已存在窗口
         let addPanelTabs = this.panels[this.activePanelIndex].tabs;
         
         let randomIndex = Math.floor(Math.random()*100);

         let newTabTitle = title?title:'覆盖tab'+ randomIndex;
         let newTabContent = content?content:'这是覆盖类型tab' + randomIndex;

         let newTab = {
                        title:newTabTitle,
                        content:newTabContent,
                        isCover:true,
                        id:this.buildIds(),
                        isShowSearch:false
                       }
         
         //遍历当前活跃窗口中是否有覆盖类型的窗口，如果有就覆盖，如果没有就添加
         let posOfCoverTab = -1;

         addPanelTabs.forEach((item,index)=>{
            if(item.isCover) {
               //有这类型的窗口,覆盖
               item = newTab;
               posOfCoverTab = index;

               this.$set(this.panels[this.activePanelIndex].tabs,index,item);
               this.$nextTick(()=>{
                  this.panels[this.activePanelIndex].activeTabIndex = index;
               })
            }
         })

         if(posOfCoverTab == -1) {
            addPanelTabs.push(newTab);
            this.panels[this.activePanelIndex].activeTabIndex = addPanelTabs.length - 1;
         }
         
         //如果不是历史记录才添加一条历史记录
         if(!title && !content) {
           this.historyPanel.unshift(newTab);
         }
       },

       //将tab分离成panel
       splitTabToPanel(index) {
          if(this.panels.length >= 3) {
            this.$message({
              showClose: true,
              message: '独立窗口数量已达到最大值！'
            })

            return;
          }
          //获取当前活跃窗口及tab，另存起来
          let activePanel = this.panels[index];
          if(activePanel.tabs.length == 1) return;

          let activeTab = activePanel.tabs[activePanel.activeTabIndex];

          let newPanel = {
              name:activeTab.title,
              tabs:[
                 {
                  title:activeTab.title,
                  content:activeTab.content,
                  isCover:activeTab.isCover,
                  id:activeTab.id,
                  isShowSearch:activeTab.isShowSearch
                 }
              ],
              activeTabIndex:0
             }

          this.panels.push(newPanel);   
          
          this.activePanelIndex = this.panels.length - 1;
          //之前对应的那一个tab被移除
          this.$nextTick(()=>{
            activePanel.tabs.splice(activePanel.activeTabIndex,1);
            activePanel.activeTabIndex = activePanel.tabs.length - 1;
          })
       },
       // 打开当前tab的文内检索
       openActiveSearch() {
          let avtivePanelTabs = this.panels[this.activePanelIndex].tabs;
          let avtiveTabIndex = this.panels[this.activePanelIndex].activeTabIndex;

          avtivePanelTabs[avtiveTabIndex].isShowSearch = !avtivePanelTabs[avtiveTabIndex].isShowSearch;

          this.$set(this.panels[this.activePanelIndex].tabs,avtiveTabIndex,avtivePanelTabs[avtiveTabIndex]);
       },

       //关闭窗口
       closeActivePanel(e,index) {
         if(index) {
           this.panels.splice(index,1);
         }else {
           this.panels.splice(this.activePanelIndex,1);
         }
         
         this.activePanelIndex = this.panels.length - 1;
       },

       //恢复历史窗口
       restoreHistoryPanel(row, event, column) {
          if(row.isCover) {
             this.addCoverTab(row.title,row.content);
          }else {
             this.addTab(row.title,row.content);
          }
          this.dialogTableVisible = false;
       },

       //删除当前tab
       closeCurTab(curTabs,tabIndex,index) {
          let preIndex = this.panels[index].activeTabIndex;

          if(curTabs.length == 1) {
             //此时只有一个tab，再点击会删除整个窗口
             this.panels.splice(index,1);
             return;
          }

          curTabs.splice(tabIndex,1);
 
          this.$set(this.panels[index],'tabs',curTabs);

          if(preIndex == curTabs.length) {
            //当前活跃tab页是最后一个
            this.panels[index].activeTabIndex = this.panels[index].tabs.length - 1;
          }
       },

       //删除当前tab外其他tab
       closeOtherTab(index) {
          let curTab = this.panels[index].tabs[this.panels[index].activeTabIndex];
          this.panels[index].tabs = [];

          this.panels[index].tabs.push(curTab);
          this.panels[index].activeTabIndex = 0;
       },

       //删除所有选项卡或删除当前外所有选项卡
       handleClose(type) {
         // closeType-1 删除所有选项卡  2 - 删除当前外所有选项卡
         let index = type.split('-')[0];
         let closeType = type.split('-')[1];

         switch (closeType) {
           case '1':
             this.closeActivePanel('',index);
             break;
           case '2':
             this.closeOtherTab(index);
             break;  
           default:
             // statements_def
             break;
         }
       },

       //设置分隔条
       resetSplit() {
           if(this.instance) {
              this.instance.destroy();
              this.instance = null;
           }

           if(this.panels.length == 1) {
             var ele = document.getElementById('panel0');

             if(ele.className.indexOf(' split split-horizontal') != -1) {
                 ele.className = ele.className.slice(0,ele.className.indexOf(' split split-horizontal'));
                 ele.style.width = this.winWidth*0.95 +'px';
                 return;
             }
           }

           if(this.panels.length == 0) return;

           switch (this.panels.length) {
             case 2:
                this.instance = Split(['#panel0', '#panel1'], {
                    size:[50,50], 
                    minSize: [180, 180],
                    gutterSize:5
                });
               break;
             case 3:
                this.instance = Split(['#panel0', '#panel1', '#panel2'], {
                    size:[33.3,33.3,33.3],
                    minSize: [180, 180, 180],
                    gutterSize:5
                });
               break;  
             default:
               // statements_def
               break;
           }

           var listOfSepetor = document.getElementsByClassName('gutter-horizontal');
           for(var i = 0 ; i < listOfSepetor.length ; i ++) {
               var a = listOfSepetor[i];
               if(listOfSepetor[i].previousSibling.className.indexOf(' split split-horizontal') == -1) {
                  listOfSepetor[i].previousSibling.className += ' split split-horizontal';
               }

               if(listOfSepetor[i].nextSibling.className.indexOf(' split split-horizontal') == -1) {
                  listOfSepetor[i].nextSibling.className += ' split split-horizontal';
               }
               listOfSepetor[i].style.height = '100%';

               listOfSepetor[i].style.cssText += `background-color: #2ab;background-repeat: no-repeat; background-position: 50%;background-image:url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAeCAYAAADkftS9AAAAIklEQVQoU2M4c+bMfxAGAgYYmwGrIIiDjrELjpo5aiZeMwF+yNnOs5KSvgAAAABJRU5ErkJggg==');cursor: ew-resize;`;
               
           }
       },

       //移动左右窗口
       startExchange(e,index) {
          //只能拖拽tab-list
          this.startMoveTimer = setTimeout(()=>{
             if(e.target.className == 'tab-list' && this.panels.length > 1) {
               this.movePanelIndex = index;

               this.movePanel = this.panels[index];
               this.isStartMove = true;
               this.parentOffsetLeft = e.target.parentNode.parentNode.offsetLeft;
               this.parentOffsetTop = e.target.parentNode.parentNode.offsetTop;
               this.movePanelLeft = e.pageX - 1/this.panels.length * this.winWidth*0.95/2;
               
             }
          },100)
       },

       overExchange(e) {
         if(this.isStartMove) {
            // 分情况讨论两窗口和三窗口
            // 先设置移动窗口的位置
            this.movePanelLeft = e.pageX - 1/this.panels.length * this.winWidth*0.95/2;
         } 
       },
       //结束移动
       endExchange(e) {
         if(this.startMoveTimer) clearTimeout(this.startMoveTimer);
         
         this.isStartMove = false;

         let panelWidth = 1/this.panels.length * this.winWidth*0.95;

         switch (this.panels.length) {
          //两窗口下
          case 2:  
               var [firstPanel,secPanel] = this.panels;
               if(this.movePanelIndex === 0) {
                  //此时拖的是第一个窗口
                  if(this.movePanelLeft >= this.parentOffsetLeft + panelWidth/2) {
                     this.panels = [secPanel,firstPanel];
                  }
               }else if(this.movePanelIndex === 1) {
                 //此时拖的是第二个窗口
                  if(this.movePanelLeft <= this.parentOffsetLeft + panelWidth/2) {
                     this.panels = [secPanel,firstPanel];
                  }
               }

            break;
          case 3:
           //三窗口下
              var [firstPanel,secPanel,thirdPanel] = this.panels;

              if(this.movePanelIndex === 0) {
                  //此时拖的是第一个窗口
                  if(this.movePanelLeft >= this.parentOffsetLeft + panelWidth/2 && this.movePanelLeft < this.parentOffsetLeft + 3*panelWidth/2) {
                     this.panels = [secPanel,firstPanel,thirdPanel];
                  }else if(this.movePanelLeft >= this.parentOffsetLeft + 3*panelWidth/2) {
                     this.panels = [thirdPanel,secPanel,firstPanel];
                  }

               }else if(this.movePanelIndex === 1) {
                 //此时拖的是第二个窗口
                  if(this.movePanelLeft <= this.parentOffsetLeft + panelWidth/2) {

                     this.panels = [secPanel,firstPanel,thirdPanel];
                  }else if(this.movePanelLeft >= this.parentOffsetLeft + 3*panelWidth/2) {
                     this.panels = [firstPanel,thirdPanel,secPanel];
                  }
               }else if(this.movePanelIndex === 2) {
                 //此时拖的是第三个窗口
                  if(this.movePanelLeft <= this.parentOffsetLeft + panelWidth/2) {

                     this.panels = [thirdPanel,secPanel,firstPanel];
                  }else if(this.movePanelLeft >= this.parentOffsetLeft + panelWidth/2 && this.movePanelLeft < this.parentOffsetLeft + 3*panelWidth/2) {
                     this.panels = [firstPanel,thirdPanel,secPanel];
                  }
               }
            break;  
          default:
            // statements_def
            break;
         }


        this.movePanelIndex = -1;  
       },

    },
    mounted() {
       window.onresize = () =>{
          this.setWindow()
       }
    },
    watch:{
       'panels.length':{
          handler:function(nv,ov) {
             if(nv !== ov && nv!== 0) {
                this.$nextTick(()=>{
                  this.resetSplit();
                })
             }
          },
          deep:true
       }
    },
    updated() {
       

    }
}
</script>

<style scoped>
.book-detail {
    overflow: hidden;
    margin: 10px auto
}
 
 .container {
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  margin-top: 10px;
  width: 98%;
  height: 90%;
  border: 1px solid #ccc;
 }

 .panel {
   position: relative;
   height: 100%;
   border-right: 1px solid #ccc;
   cursor: pointer;
 }

 .move-panel {
   position: fixed;
   z-index: 999;
   height: 90%;
   top: 0;
   background: #eee;
   transform: all 1s;
 }

 .active-panel {
   background: #eee;
 }

 .tab-list {
   display: flex;
   justify-content: space-between;
   align-items: center;
   position: relative;
   height: 50px;
   border-bottom: 1px solid #ccc;
 }


 .tab-cont {
   display: flex;
   justify-content: flex-start;
   align-items: center;
   height: 100%;
   /*max-width:95%; */
 }

 .tab {
   display: flex;
   justify-content: center;
   align-items: center;
   position: relative;
   width: 120px;
   height: 100%;
   overflow: hidden;
   text-overflow: ellipsis;
   white-space: nowrap;
   border-right: 1px solid #ccc;
 }

 .close-btn {
   position: absolute;
   right: 4px;
   top: 50%;
   margin-top: -8px;
   display: flex;
   justify-content: center;
   align-items: center;
   width: 16px;
   height: 16px;
   border-radius: 50%;
   background: transparent;
   display: none
 }



 .tab:hover {
   background: #ccc;
   color:#666;
 }

 .tab:hover .close-btn {
   display: flex;
 }

  .close-btn:hover {
    background: white
  }


  .panel-control {
    display: flex;
    justify-content: space-around;
    align-items: center;
    position: absolute;
    right: 0;
    top: 0;
    z-index: 10;
    width: 60px;
    height: 100%;
  }

 .panel-content {
   display: flex;
   justify-content: space-between;
   flex-direction: column;
   align-content: space-between;
   width: 100%;
   height: 94%;
   background: transparent;
 }


 .panel-separator {
   position: absolute;
   top: 0;
   width: 3px;
   height: 100%;
   background: transparent;
   cursor: col-resize
 }


 .panel-search {
/*   position: absolute;
   bottom: 0;
   left: 0;*/
   display: flex;
   justify-content: center;
   align-items: center;
   height: 40%;
   width: 100%;
   background: #ccc
 }


 .gutter {
    background-color: #eee;

    background-repeat: no-repeat;
    background-position: 50%;
}

.gutter.gutter-horizontal {
   background-image:url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAB4AAAAFAQMAAABo7865AAAABlBMVEVHcEzMzMzyAv2sAAAAAXRSTlMAQObYZgAAABBJREFUeF5jOAMEEAIEEFwAn3kMwcB6I2AAAAAASUVORK5CYII=');
    cursor: ew-resize;
}

.gutter.gutter-vertical {
    background-image: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAeCAYAAADkftS9AAAAIklEQVQoU2M4c+bMfxAGAgYYmwGrIIiDjrELjpo5aiZeMwF+yNnOs5KSvgAAAABJRU5ErkJggg==');
    cursor: ns-resize;
}

.split {
  -webkit-box-sizing: border-box;
     -moz-box-sizing: border-box;
          box-sizing: border-box;
}

.split, .gutter.gutter-horizontal {
    float: left;
}
/*
.split, .gutter.gutter-horizontal {
    height: 500px;
}
*/
.split {
    overflow-y: auto;
    overflow-x: hidden;
}

</style>
