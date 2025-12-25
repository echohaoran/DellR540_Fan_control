# 戴尔服务器r540风扇控制脚本

> iDRAC8无法细致调整风扇转速，导致在家用的时候噪音极大。所以可以使用IPMI进行管理。

## 配置
戴尔r540
系统：PVE（同理各种Linux）

## ipmitool基础语法
```shell
ipmitool -I lanplus -H <iDRAC_IP> -U <username> -P <password> raw 0x30 0x30 0x02 0x00 <speed>
```
---

因为ipmitool进行风扇管理时，需要使用十六进制，以下为对应关系：


｜转速百分比｜十六进制｜
｜---------｜---------｜
｜0%｜0x00|
｜10%｜0x0A|
｜20%｜0x14|
｜30%｜0x1E|
｜40%｜0x28|
｜50%｜0x32|
｜60%｜0x3C|
｜70%｜0x46|
｜80%｜0x50|
｜90%｜0x5A|
｜100%｜0x64|

### 示例：
```shell
# 启用手动模式
ipmitool -I lanplus -H 192.168.1.100 -U root -P calvin raw 0x30 0x30 0x01 0x00

# 设置转速为 35%（十进制 35 = 0x23）
ipmitool -I lanplus -H 192.168.1.100 -U root -P calvin raw 0x30 0x30 0x02 0x00 0x23
```

### 恢复自动风扇控制

```shell
ipmitool -I lanplus -H <iDRAC_IP> -U <user> -P <pass> raw 0x30 0x30 0x01 0x01
```


## 完整脚本带控速，需要修改转速和温度关系自己修改即可
> 记得修改文中iDRAC的ip、用户名、密码

```shell
#!/bin/bash
################################################################################
# 脚本名称：fan_control.sh
# 脚本用途：PVE环境下基于CPU温度自动调节服务器风扇转速
# 适用场景：适配你的服务器IPMI输出（Inlet Temp/Temp 3.1/Temp 3.2）
# 依赖工具：ipmitool（脚本会自动检测并安装）
# 调速规则（已修改）：
#   - CPU温度 < 50℃：风扇转速5%
#   - 50℃ ≤ CPU温度 < 70℃：风扇转速15%
#   - CPU温度 ≥ 70℃：风扇转速40%
# 检测间隔：10秒
# 日志路径：/var/log/fan_control.log
# 适配修改：针对你的IPMI输出格式优化温度提取逻辑
################################################################################

#######################################
# 【配置项】- 无需修改（已适配你的环境）
#######################################
IPMI_HOST="192.168.0.120"       # IDRAC/IPMI管理地址
IPMI_USER="root"                # IPMI登录用户名
IPMI_PASS="calvin"              # IPMI登录密码
CHECK_INTERVAL=10               # 温度检测/调速间隔（单位：秒）
LOG_FILE="/var/log/fan_control.log"  # 脚本运行日志文件路径
TEMP_KEYWORD="Temp"             # 匹配温度传感器的关键词

#######################################
# 函数：安装依赖工具（ipmitool）
# 功能：检测系统是否安装ipmitool，未安装则自动apt安装
# 入参：无
# 出参：无
#######################################
install_deps() {
    # 检查ipmitool是否已安装
    if ! command -v ipmitool &> /dev/null; then
        echo "$(date +'%Y-%m-%d %H:%M:%S') [INFO] 未检测到ipmitool工具，开始自动安装..." >> $LOG_FILE
        apt update && apt install -y ipmitool &>> $LOG_FILE
        if [ $? -ne 0 ]; then
            echo "$(date +'%Y-%m-%d %H:%M:%S') [ERROR] ipmitool安装失败，请手动执行 'apt install ipmitool' 安装！" >> $LOG_FILE
            exit 1
        fi
        echo "$(date +'%Y-%m-%d %H:%M:%S') [INFO] ipmitool安装成功" >> $LOG_FILE
    fi
}

#######################################
# 函数：关闭风扇自动调速
# 功能：通过IPMI命令禁用服务器默认的风扇自动调速功能
# 入参：无
# 出参：无
#######################################
disable_auto_fan() {
    echo "$(date +'%Y-%m-%d %H:%M:%S') [INFO] 执行关闭风扇自动调速命令..." >> $LOG_FILE
    ipmitool -I lanplus -H $IPMI_HOST -U $IPMI_USER -P $IPMI_PASS raw 0x30 0x30 0x01 0x00 &>> $LOG_FILE
    if [ $? -ne 0 ]; then
        echo "$(date +'%Y-%m-%d %H:%M:%S') [ERROR] 关闭风扇自动调速失败！请检查IPMI地址/账号/密码是否正确" >> $LOG_FILE
    else
        echo "$(date +'%Y-%m-%d %H:%M:%S') [INFO] 关闭风扇自动调速成功" >> $LOG_FILE
    fi
}

#######################################
# 函数：设置风扇转速
# 功能：将百分比转速转换为IPMI识别的十进制值，并执行调速命令
# 入参：$1 - 目标转速百分比（如5、15、40）
# 出参：无
#######################################
set_fan_speed() {
    local speed=$1  # 接收入参：转速百分比
    # 转速换算逻辑：IPMI的转速值范围是0-255（0xff=100%）
    local dec_speed=$((speed * 255 / 100))  # 十进制值（5%→12，15%→38，40%→102）
    local hex_speed=$(printf "0x%02x" $dec_speed)  # 转16进制
    echo "$(date +'%Y-%m-%d %H:%M:%S') [INFO] 开始设置风扇转速：${speed}%（十进制：${dec_speed} → 16进制：${hex_speed}）" >> $LOG_FILE
    
    # 执行IPMI调速命令
    ipmitool -I lanplus -H $IPMI_HOST -U $IPMI_USER -P $IPMI_PASS raw 0x30 0x30 0x02 0xff $dec_speed &>> $LOG_FILE
    
    # 检查调速结果
    if [ $? -ne 0 ]; then
        echo "$(date +'%Y-%m-%d %H:%M:%S') [ERROR] 设置风扇转速${speed}%失败！" >> $LOG_FILE
    else
        echo "$(date +'%Y-%m-%d %H:%M:%S') [INFO] 设置风扇转速${speed}%成功" >> $LOG_FILE
    fi
}

#######################################
# 函数：获取CPU温度（通用适配版）
# 功能：通过"degrees"分隔提取温度，彻底解决列数问题
# 入参：无
# 出参：返回温度数值；获取失败返回-1
#######################################
get_package_temp() {
    # 执行IPMI温度查询，保存原始输出
    local ipmi_output=$(ipmitool -I lanplus -H $IPMI_HOST -U $IPMI_USER -P $IPMI_PASS sdr type temperature 2>/dev/null)
    # 容错：检查IPMI命令是否执行成功
    if [ $? -ne 0 ]; then
        echo "$(date +'%Y-%m-%d %H:%M:%S') [ERROR] IPMI连接失败！请检查192.168.0.120是否可达、账号密码是否正确" >> $LOG_FILE
        echo -1
        return
    fi
    # 通用提取逻辑：按"degrees"分割行，取分割前的最后一个数字（温度值）
    # 步骤1：过滤掉进风口温度，只保留CPU Temp行
    local cpu_temp_lines=$(echo "$ipmi_output" | grep -i "$TEMP_KEYWORD" | grep -v "Inlet")
    # 步骤2：取最后一行（3.2的Temp），按"degrees"分割，取前面部分的最后一个数字
    local temp_raw=$(echo "$cpu_temp_lines" | tail -1 | awk -F 'degrees' '{print $1}' | awk '{print $NF}')
    
    # 容错：检查温度值是否为有效数字
    if [ -z "$temp_raw" ] || ! [[ "$temp_raw" =~ ^[0-9]+$ ]]; then
        echo "$(date +'%Y-%m-%d %H:%M:%S') [ERROR] 温度获取失败！IPMI原始输出：$ipmi_output" >> $LOG_FILE
        echo -1
        return
    fi
    # 返回有效温度值
    echo $temp_raw
}

#######################################
# 主函数：脚本核心逻辑
# 流程：初始化（安装依赖→关闭自动调速）→ 循环检测温度→按规则调速
#######################################
main() {
    # 确保日志文件存在
    touch $LOG_FILE
    
    # 第一步：初始化操作
    echo "$(date +'%Y-%m-%d %H:%M:%S') [INFO] 风扇自动调速脚本启动..." >> $LOG_FILE
    install_deps          # 安装依赖工具
    disable_auto_fan      # 关闭自动调速
    
    # 第二步：无限循环执行温度检测和调速
    while true; do
        # 获取当前CPU温度
        local pg_temp=$(get_package_temp)
        
        # 温度获取失败：等待间隔后重试
        if [ "$pg_temp" -eq -1 ]; then
            sleep $CHECK_INTERVAL
            continue
        fi
        
        # 记录当前温度
        echo "$(date +'%Y-%m-%d %H:%M:%S') [INFO] 当前CPU温度：${pg_temp}°C" >> $LOG_FILE
        
        # 按修改后的规则执行调速
        if [ "$pg_temp" -lt 50 ]; then
            # 温度 < 50℃ → 5%转速（核心修改点）
            set_fan_speed 5
        elif [ "$pg_temp" -ge 50 ] && [ "$pg_temp" -lt 70 ]; then
            # 50℃ ≤ 温度 < 70℃ → 15%转速
            set_fan_speed 15
        elif [ "$pg_temp" -ge 70 ]; then
            # 温度 ≥ 70℃ → 40%转速
            set_fan_speed 40
        fi
        
        # 等待检测间隔后，进入下一次循环
        sleep $CHECK_INTERVAL
    done
}

#######################################
# 脚本入口：启动主函数
#######################################
main
```




