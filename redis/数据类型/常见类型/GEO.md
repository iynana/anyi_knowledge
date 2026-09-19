+ 附件的商家（存储商家自身的经纬度``GEOADD key 经度 维度 商户ID``，调用``GEORADIUS``查询以用户位置为中心，指定半径查询附件商户）

GEO先基于ZSet利用score粗筛，后通过Haversine精确计算

