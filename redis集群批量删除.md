# Redis Cluster  linux下批量删除键

## 说明

使用时不支持传入参数 *， 如 redis_batch_del.sh *， 因为在linux下 会自动将 * 解析为当前目录下所有文件名， 目前还没有想到好的解决办法。

如果需要flushall 可以自行加入参数判断， 执行flushall。

## 代码

    #!/usr/bin/env bash

    ###############################################################################################
    #使用说明: 
    #  脚本需要设置 三个参数, 即:
    #     1. redis_home, 注意, 最后没有 /
    #     2. redis_password
    #     3. redis_enable_port redis集群中任意可用节点的端口
    #   参数设置完毕后, 执行脚本时, 可同时删除多个key, 匹配方式为 redis的 pattern.
    #	多个参数之间以空格隔开. 
    #   在任意redisCluster的节点上执行即可
    #   如 redis_batch_del.sh hello* hhh* *hhh*	
    ###############################################################################################

    redis_home='/opt/TDS/redis/'

    #redis密码
    redis_password='redis'

    #redis 任意可用redis 服务端端口
    redis_enable_port='7001'

    #如果没有redis密码， 需要删除 -a $reis_password
    master_info=(`$redis_home/src/redis-cli -c -a $redis_password -p $redis_enable_port cluster nodes |grep 'master'`)

    address_array=()
    #获取所有master节点
    for item in ${master_info[@]}
    do
        if [[ $item =~ '@' ]]
        then
            echo '当前redis的 master 节点为:' $item
            address_array[${#address_array[*]}]=$item
        fi
    done

    #最终addresses 存储的为 ip port, 例 ${addresses[0]} 为 192.168.0.1 ${addresses[1]} 为 7001
    addresses=()
    for item in ${address_array[@]}
    do
        temp_split=(${item//@/ })
        ip_port=${temp_split[0]}
        temp_split=(${ip_port//:/ })
        addresses[${#addresses[*]}]=${temp_split[0]}
        addresses[${#addresses[*]}]=${temp_split[1]}
    done

    function del_key_with_pattern() {

        local length=${#addresses[*]}
        local index=0
        while [ $index -lt $length ]
        do
            local port_index=`expr $index + 1`
            #如果没有redis密码， 需要删除 -a $reis_password
            local redis_command="$redis_home/src/redis-cli -a $redis_password -c -h ${addresses[${index}]} -p ${addresses[${port_index}]}"
            #屏蔽错误信息.
            $redis_command keys $1 2>/dev/null |xargs -i $redis_command del {} >/dev/null 2>&1
            echo "清除节点: ${addresses[${index}]}:${addresses[${port_index}]} 的 $1 数据"
            index=`expr $port_index + 1`
        done
    }

    for item in "$@"
    do
        del_key_with_pattern $item
    done
	
