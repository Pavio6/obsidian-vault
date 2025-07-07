**使用流程**：
1. 定义 .proto 文件
2. 使用protoc编译器生成代码
3. 将生成的代码编译到项目中
4. 使用protocol buffer 序列化与反序列化

#### 命令一
protoc --go_out=. --go_opt paths=source_relative .\echo\echo.proto
- --go_out=. ：指定go代码输出目录， . 代表当前目录，生成的代码会放在与.proto文件相同的目录结构下
- --go_opt paths=source_relative：控制生成文件的路径规则，paths=source_relative代表生成的文件放在.proto文件相同的目录下
- .\echo\echo.proto：指定要编译的.proto文件路径
#### 命令二
 protoc --go-grpc_out=. --go-grpc_opt paths=source_relative .\echo\echo.proto
 **使用 Protocol Buffers (protobuf) 的编译器 `protoc` 生成 gRPC 服务相关的 Go 代码**