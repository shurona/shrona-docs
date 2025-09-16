---
tags:
  - mybatis
---
``` xml
<select id="findAllByOrderSeqs" resultType="kr.co.meatmatch.api.importSales.sales.infrastructure.entity.ImportSalesEntity">
    SELECT *
    FROM tb_import_sales
    WHERE delete_yn = 'N'
    <if test="orderSeqList != null and orderSeqList.size() > 0">
        AND import_sales_order_seq IN
        <foreach collection="orderSeqList" item="orderSeq" open="(" close=")" separator=",">
            #{orderSeq}
        </foreach>
    </if>
</select>
```
