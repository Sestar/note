# poi
</br>

## 添加下拉框
</br>

### 主要思路

```test
    需要将下拉框选项存放在另一个Sheet(别名dropDownSheet)中, 
    将显示Sheet(别名dataSheet)下拉框绑定到dropDownSheet对应部分。并将dropDownSheet隐藏
```

</br>

### 代码部分

```Java
    /**
     * @description Excel生成下拉框
     * @author zhangxinxin
     * @param tarSheet 显示Sheet
     * @param hideSheet 下拉框选项存储的Sheet
     * @param menuItems 下拉框数据
     * @param menuItemNbrs 下拉框数据对应数值
     * @param columnOfHide 下拉框数据对应隐藏Sheet的列数
     * @param title 下拉框表头
     * @param columnOfData 下拉框在数据Excel中列数
     * @param titleCellStyle 表头单元格样式
     **/
    public static void genDropDown(Sheet tarSheet, Sheet hideSheet, List<String> menuItems, 
                    List<String> menuItemNbrs, int columnOfHide, String title, int columnOfData, 
                    CellStyle titleCellStyle) {
        Row row;
        Cell cell;
        // 设置标题
        row = getRow(0, hideSheet);
        cell = getCell(columnOfHide, row);
        cell.setCellValue(title);
        cell.setCellStyle(titleCellStyle);

        // 下拉框内容
        for (int i = 1; i <= menuItems.size(); i++) {
            row = getRow(i, hideSheet);
            // 文字描述部分
            hideSheet.setColumnWidth(columnOfHide, 10 * 512);
            cell = getCell(columnOfHide, row);
            cell.setCellValue(menuItems.get(i-1));
            // 主数据码部分
            hideSheet.setColumnWidth(columnOfHide+1, 10 * 512);
            cell = getCell(columnOfHide+1, row);
            cell.setCellValue(menuItemNbrs.get(i-1));
        }
        CellRangeAddressList addressList;
        DataValidation dataValidation;
        row = getRow(1, tarSheet);
        cell = getCell(columnOfData, row);
        // 下拉框默认为第一个
        cell.setCellValue(menuItems.get(0));
        addressList = new CellRangeAddressList(1, 1, columnOfData, columnOfData);
        dataValidation = getDataValidation(hideSheet, menuItems.size(), addressList, columnOfHide);
        tarSheet.addValidationData(dataValidation);
        // 最后在外部设置hideSheet隐藏 workbook.setSheetHidden(idxOfHideSheetworkbook, true);
    }
```
```Java
    /**
     * @description 下拉框内容设置关联
     * @author zhangxinxin
     * @param hideSheet 数据Sheet
     * @param menuItemSize 下拉框内容数量
     * @param addressList 下拉对应数据Sheet行列
     * @param columnOfHide 下拉框Sheet列数
     * @return org.apache.poi.ss.usermodel.DataValidation
     **/
    private static DataValidation getDataValidation(Sheet hideSheet, int menuItemSize,
                                                    CellRangeAddressList addressList, int columnOfHide) {
        DataValidation validation;
        // 绑定数据规则(必须要加上$,否则数据规则会出现不可预计的错误，比如点击的不同单元格, 规定绑定行数会发生改变)
        String letterOfColumn = getLetterOfColumn(columnOfHide);
        String formula = hideSheet.getSheetName() + 
            "!$"+ letterOfColumn +"$2:$"+ letterOfColumn + "$" + (menuItemSize+1);
        // 通过关联规则创建DataValidation
        if (HSSFSheet.class.equals(hideSheet.getClass())) {
            // 使用createFormulaListConstraint而不使用createExplicitListConstraint, 因为explicit生成的
            // 下拉框内容有长度的限制, 是直接将数据赋值到下拉框中, 而formula却是将下拉框内容通过规则绑定,
            // 例如这里就是和hideSheet的!$A$2:$A$32进行关联
            DVConstraint dvConstraint = DVConstraint.createFormulaListConstraint(formula);
            validation = new HSSFDataValidation(addressList, dvConstraint);
        } else if (XSSFSheet.class.equals(hideSheet.getClass())) {
            XSSFDataValidationHelper dvHelper = new XSSFDataValidationHelper((XSSFSheet) hideSheet);
            XSSFDataValidationConstraint dvConstraint = (XSSFDataValidationConstraint) dvHelper.createFormulaListConstraint(formula);
            validation = dvHelper.createValidation(dvConstraint, addressList);
        } else {
            throw ErrorCodeHelper.PARAM_ERROR_CODE(BatchOrderErrorEnum.EXCEL_TYPE_ERROR).toBizException();
        }
        // 输入非法数据时，弹窗警告框(下拉框内容不能为空)
        validation.setEmptyCellAllowed(false);
        validation.setShowErrorBox(true);
        return validation;
    }
```
```Java
    /**
     * @description 返回Excel列数对应字母
     * @author zhangxinxin
     * @param column 列数
     * @return java.lang.String
     **/
    private static String getLetterOfColumn(int column) {
        if (column < 0) {
            return null;
        }
        // 先将column转成26进制,再填充对应字母
        char letter;
        int binary = 26;
        StringBuilder letterOfColumn = new StringBuilder();
        do {
            letter = 'A';
            letter += column % binary;
            letterOfColumn.insert(0, letter);
            column /= binary;
        } while (column == 0);
        return letterOfColumn.toString();
    }
```
```Java
/**
     * @description 获取Row
     * @author zhangxinxin
     * @param rowNum 行数
     * @param sheet 目标Sheet
     * @return org.apache.poi.ss.usermodel.Row
     **/
    private static Row getRow(int rowNum, Sheet sheet) {
        Row row = sheet.getRow(rowNum);
        if (null == row) {
            row = sheet.createRow(rowNum);
        }
        return row;
    }

    /**
     * @description 获取Cell
     * @author zhangxinxin
     * @param cellNum 列数
     * @param row 行对象
     * @return org.apache.poi.ss.usermodel.Cell
     **/
    private static Cell getCell(int cellNum, Row row) {
        Cell cell = row.getCell(cellNum);
        if (null == cell) {
            cell = row.createCell(cellNum);
        }
        return cell;
    }
```