hashCode()默认和地址相关
+ 存储在HashMap、HashSet中时，首先需要使用hashCode()确定对象的存储桶，然后在桶内使用equals()进行精确比较。