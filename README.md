# Claude-code-analysis
  ---                                                                                                            
  报告内容概览                                                                                                   
                                                                                                                 
  报告共 8 章，覆盖 1904 个源文件，主要内容：                                                                    
                                                                                                                 
  第 1-2 章：项目概览与整体架构                                                                                  
  - 技术栈（TypeScript + Bun + React/Ink）                                                                       
  - 8 层纵向架构图（从 UI 到基础设施）                                                                           
                                                                                                               
  第 3 章：10 个子系统详解                                                                                       
  - 启动层：三阶段并行预取优化（MDM + Keychain + 功能门控）                                                      
  - 引擎层：QueryEngine.ts 的完整 AI 交互生命周期                                                                
  - 工具系统：42 个工具分类表（文件/执行/智能/任务/MCP/工作流/多Agent）                                          
  - 命令系统：101 个命令的领域分组                                                                               
  - 状态管理：DeepImmutable 不可变状态树                                                                         
  - UI 层：自定义 Ink 终端渲染引擎（React → ANSI 的完整管线）                                                    
  - 服务层：20 个服务（API/MCP/OAuth/LSP/记忆/压缩等）                                                           
  - Bridge 架构：本地↔云端 CCR 双向通信协议                                                                      
  - 多 Agent 系统：子 Agent 状态隔离机制                                                                         
  - Plugin/Skill 系统：三层扩展机制                                                                              
                                                                                                                 
  第 4 章：数据流分析                                                                                            
  - 用户消息完整处理流程（6 步）                                                                                 
  - 工具权限决策树                                                                                               
  - Bridge 模式通信时序                                                                                          
                                                                                                                 
  第 5-8 章：设计模式、功能门控、扩展点地图、关键文件速查表     
