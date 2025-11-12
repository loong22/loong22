### 执行脚本

```
#!/bin/bash

# 设置S3桶的名称和目标HTML文件名 
BUCKET_NAME="bucket-name"
OUTPUT_FILE="/opt/TMP/index.html"

# 使用rclone列出S3桶中的所有对象
OBJECTS=$(rclone lsjson s3:$BUCKET_NAME --recursive)

# 开始生成HTML文件
cat > $OUTPUT_FILE << 'EOF'
<!DOCTYPE html>
<html lang='zh-cn'>
<head>
    <meta charset='UTF-8'>
    <title>对象列表</title>
    <style>
        table { border-collapse: collapse; width: 100%; }
        th, td { padding: 8px; text-align: left; border-bottom: 1px solid #ddd; }
        th { background-color: #f2f2f2; }
        tr:hover { background-color: #f5f5f5; }
        .size { text-align: right; }
        .date { text-align: center; }
    </style>
</head>
<body>
    <h1>对象列表</h1>
    <table>
        <tr>
            <th>文件名</th>
            <th>大小</th>
            <th>修改时间</th>
        </tr>
EOF

# 定义函数用于格式化文件大小
format_size() {
    local size=$1
    if [ $size -ge 1073741824 ]; then
        printf "%.1f GB" $(echo "scale=1; $size/1073741824" | bc)
    elif [ $size -ge 1048576 ]; then
        printf "%.1f MB" $(echo "scale=1; $size/1048576" | bc)
    elif [ $size -ge 1024 ]; then
        printf "%.1f KB" $(echo "scale=1; $size/1024" | bc)
    else
        echo "$size B"
    fi
}

# 解析JSON并生成带有大小和修改时间的表格行
echo $OBJECTS | jq -r '.[] | select(.IsDir==false) | [.Path, .Size, .ModTime] | @tsv' | while IFS=$'\t' read -r path size modtime; do
    formatted_size=$(format_size $size)
    formatted_date=$(date -d "$modtime" '+%Y-%m-%d %H:%M:%S')
    echo "<tr>" >> $OUTPUT_FILE
    echo "<td><a href=\"https://example.com/$path\">$path</a></td>" >> $OUTPUT_FILE
    echo "<td class=\"size\">$formatted_size</td>" >> $OUTPUT_FILE
    echo "<td class=\"date\">$formatted_date</td>" >> $OUTPUT_FILE
    echo "</tr>" >> $OUTPUT_FILE
done

# 完成HTML文件
echo "</table></body></html>" >> $OUTPUT_FILE

echo "HTML文件已生成: $OUTPUT_FILE"

# 上传生成的HTML文件到S3
aws s3 cp ./index.html s3://example/index.html --endpoint-url https://example.com
```