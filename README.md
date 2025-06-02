# 7zAutomator
 7z使用自动操作加入右键
 ![用法](https://github.com/Marspacecraft/7zAutomator/blob/main/pic2.png)   


 ## 自动操作设置  

* ```brew install p7zip```安装7z
* 快速操作->实用工具->运行Shell脚本
 
![用法](https://github.com/Marspacecraft/7zAutomator/blob/main/pic.png)   



## 脚本内容  
* 成功播放```afplay /System/Library/Sounds/Glass.aiff```(使用iterm试听)
* 失败播放```afplay /System/Library/Sounds/Ping.aiff```(使用iterm试听)或弹出运行异常窗口
* 如果运行不成功尝试修改```/opt/homebrew/bin/7z```路径为自己环境的路径
  

 ### 7z解压脚本  

* 解压缩，如果不支持的格式会出现告警   

![用法](https://github.com/Marspacecraft/7zAutomator/blob/main/pic1.png)   

```shell
# 设置错误处理：任何命令失败立即退出
set -e

# 支持的压缩格式列表（7z支持的解压格式）
supported_formats=(
    "7z" "zip" "rar" "tar" "gz" "bz2" "xz" 
    "zst" "lzma" "cab" "arj" "lzh" "chm" 
    "msi" "dmg" "iso" "cpio" "rpm" "deb"
    "vhd" "wim" "apk" "jar" "xar" "z" 
    "001" "split" "exe"
)

# 处理每个选中的压缩文件
for archive in "$@"; do
    # 获取文件扩展名（小写）
    extension="${archive##*.}"
    extension_lower=$(echo "$extension" | tr '[:upper:]' '[:lower:]')
    
    # 验证是否为支持的压缩格式
    if [[ ! " ${supported_formats[@]} " =~ " ${extension_lower} " ]]; then
		afplay /System/Library/Sounds/Ping.aiff
        osascript -e "display alert \"不支持的文件类型\" message \"${archive} 不是支持的压缩格式 (.${extension})\""
        exit 0
    fi

    # 获取文件信息
    archive_name=$(basename "$archive")
    base_name="${archive_name%.*}"
    parent_dir=$(dirname "$archive")
    tmp_dir=$(mktemp -d)
    
    # 特殊处理多卷压缩文件
    if [[ "$extension_lower" == "001" || "$extension_lower" == "split" ]]; then
        # 查找同系列文件
        archive_base="${archive%.*}"
        archive_series=("$archive_base".*)
        
        # 解压多卷文件
        /opt/homebrew/bin/7z x -o"$tmp_dir" "${archive_series[1]}" > /dev/null
    else
        # 解压单个文件
        /opt/homebrew/bin/7z x -o"$tmp_dir" "$archive" > /dev/null
    fi
    
    # 检查解压内容
    extracted_items=("$tmp_dir"/*)
    if [ ${#extracted_items[@]} -eq 0 ]; then
		afplay /System/Library/Sounds/Ping.aiff
        osascript -e "display alert \"解压失败\" message \"${archive_name} 可能为空或损坏\""
        rm -rf "$tmp_dir"
        exit 0
    fi
    
    # 处理解压内容
    for item in "$tmp_dir"/*; do
        item_name=$(basename "$item")
        target_path="$parent_dir/$item_name"
        
        # 处理重名
        counter=1
        while [[ -e "$target_path" ]]; do
            # 分离文件名和扩展名
            filename="${item_name%.*}"
            extension="${item_name##*.}"
            
            # 特殊处理无扩展名文件
            if [[ "$filename" == "$extension" ]]; then
                target_path="$parent_dir/${item_name} ${counter}"
            else
                # 处理带扩展名文件
                if [[ $counter -eq 1 ]]; then
                    target_path="$parent_dir/${filename} ${counter}.${extension}"
                else
                    # 移除之前的计数器
                    new_filename="${filename% [0-9]*}"
                    target_path="$parent_dir/${new_filename} ${counter}.${extension}"
                fi
            fi
            ((counter++))
        done
        
        # 移动到目标位置
        mv -n "$item" "$target_path"
    done
    
    # 清理临时目录
    rm -rf "$tmp_dir"
done

# 完成提示
afplay /System/Library/Sounds/Glass.aiff
exit 0
```

### 7z压缩脚本  

* 压缩，如果出现重名会告警并重新命名   

![用法](https://github.com/Marspacecraft/7zAutomator/blob/main/pic3.png)   

```shell
# 设置错误处理：任何命令失败立即退出
set -e

# 处理每个选中项目
for item in "$@"; do
    # 获取基础名称和父路径
    name=$(basename "$item")
    parent_dir=$(dirname "$item")
    
    # 创建临时目录
    tmp_dir=$(mktemp -d)
    tmp_7z="${tmp_dir}/${name}.tmp.7z"
    
    # 压缩到临时文件
    /opt/homebrew/bin/7z a -t7z "$tmp_7z" "$item" > /dev/null
    
    # 确定最终文件名（避免覆盖）
    base_name="${name}"
    counter=1
    final_7z="${parent_dir}/${base_name}.7z"
    
    # 检查文件是否存在并生成新名称
    while [[ -e "${final_7z}" ]]; do
        if [[ $counter -eq 1 ]]; then
            final_7z="${parent_dir}/${base_name} ${counter}.7z"
        else
            final_7z="${parent_dir}/${base_name} ${counter}.7z"
        fi
        ((counter++))
    done
	
	# 重命名告警
	if [[ $counter -ne 1 ]]; then
        osascript -e "display alert \"存在同名文件\" message \"重命名为${final_7z} \""
	fi
    
    # 移动到最终位置
    mv -n "$tmp_7z" "$final_7z"
    
    # 清理临时目录
    rm -rf "$tmp_dir"
done

# 完成提示
afplay /System/Library/Sounds/Glass.aiff
```
