# 查找当前文件夹中所有以 .proto 结尾的文件
file(GLOB PROTO_FILES "*.proto")

set(GENERATED_OUT_PATH ${PROJECT_SOURCE_DIR}/generated)
include_directories(${GENERATED_OUT_PATH})

# 查找grpc相关
find_package(Protobuf CONFIG REQUIRED)
find_package(absl CONFIG REQUIRED)
find_package(gRPC CONFIG REQUIRED)

#查找本地Protobuf模块的库信息，实际上CMake就是在找Protobuf-config.cmake文件
set(protobuf_MODULE_COMPATIBLE TRUE)
find_package(Protobuf CONFIG REQUIRED)
message(STATUS "Using protobuf ${Protobuf_VERSION}")

set(_PROTOBUF_LIBPROTOBUF protobuf::libprotobuf)
set(_REFLECTION gRPC::grpc++_reflection)
if(CMAKE_CROSSCOMPILING)
    find_program(_PROTOBUF_PROTOC protoc)
else()
    set(_PROTOBUF_PROTOC $<TARGET_FILE:protobuf::protoc>)
endif()

# Find gRPC installation
# Looks for gRPCConfig.cmake file installed by gRPC's cmake installation.
#查找本地gRPC模块的库信息
find_package(gRPC CONFIG REQUIRED)
message(STATUS "Using gRPC ${gRPC_VERSION}")

set(_GRPC_GRPCPP gRPC::grpc++)
if(CMAKE_CROSSCOMPILING)
    find_program(_GRPC_CPP_PLUGIN_EXECUTABLE grpc_cpp_plugin)
else()
    set(_GRPC_CPP_PLUGIN_EXECUTABLE $<TARGET_FILE:gRPC::grpc_cpp_plugin>)
endif()

# 遍历所有的proto文件
foreach(data_proto ${PROTO_FILES})
    # 去除后缀
    get_filename_component(proto ${data_proto} NAME_WE)
    # 打印
    message("current generating: ${proto}")
    # 创建生成代码的文件夹
    file(MAKE_DIRECTORY "${GENERATED_OUT_PATH}/protos/${proto}")
    # 设置4个proto生成代码文件
    set(data_proto_srcs "${GENERATED_OUT_PATH}/protos/${proto}/${proto}.pb.cc")
    set(data_proto_hdrs "${GENERATED_OUT_PATH}/protos/${proto}/${proto}.pb.h")
    set(data_grpc_srcs "${GENERATED_OUT_PATH}/protos/${proto}/${proto}.grpc.pb.cc")
    set(data_grpc_hdrs "${GENERATED_OUT_PATH}/protos/${proto}/${proto}.grpc.pb.h")

    ## 生成文件
    add_custom_command(
        OUTPUT "${data_proto_srcs}" "${data_proto_hdrs}" "${data_grpc_srcs}" "${data_grpc_hdrs}"
        COMMAND protoc
        ARGS --grpc_out "${GENERATED_OUT_PATH}/protos/${proto}"
            --cpp_out "${GENERATED_OUT_PATH}/protos/${proto}"
            -I "${PROJECT_SOURCE_DIR}/resources/protos"
            --plugin=protoc-gen-grpc=`which grpc_cpp_plugin`
            "${data_proto}"
        DEPENDS "${data_proto}")
    
    #将grpc的四个文件编译成一个库
    # data_grpc_proto
    message(${proto}_data_grpc_proto)
    add_library(${proto}_data_grpc_proto
        ${data_grpc_srcs}
        ${data_grpc_hdrs}
        ${data_proto_srcs}
        ${data_proto_hdrs})
    target_include_directories(${proto}_data_grpc_proto PUBLIC
        ${GENERATED_OUT_PATH}/protos
    )
    #链接库data_grpc_proto，会生成链接文件
    target_link_libraries(${proto}_data_grpc_proto PUBLIC
        ${_REFLECTION}
        ${_GRPC_GRPCPP}
        ${_PROTOBUF_LIBPROTOBUF})
endforeach()
