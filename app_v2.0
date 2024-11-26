import streamlit as st
import pdfplumber
import pandas as pd
import os
import tempfile

def cleanup_temp_files(file_paths):
    """清理临时文件"""
    for file_path in file_paths:
        try:
            os.remove(file_path)
        except Exception as e:
            print(f"删除文件 {file_path} 时出错: {e}")

def process_pdfs(pdf_files, file_names):
    """处理上传的 PDF 文件，提取表格并返回处理后的 DataFrame"""
    all_data = []
    no_table_files = []

    for pdf_path, original_name in zip(pdf_files, file_names):
        try:
            with pdfplumber.open(pdf_path) as pdf:
                data = []
                for page in pdf.pages:
                    tables = page.extract_tables()
                    if tables:
                        for table in tables:
                            for row in table:
                                cleaned_row = [cell.replace('\n', '') if isinstance(cell, str) else cell for cell in row]
                                data.append(cleaned_row)

                df = pd.DataFrame(data)
                # 查找“汇总”行的位置
                start_indices = df[df.apply(lambda row: row.astype(str).str.contains('汇总').any(), axis=1)].index
                # 查找“报关单草稿”行的位置
                end_indices = df[df.apply(lambda row: row.astype(str).str.contains('报关单草稿').any(), axis=1)].index
                if end_indices.size == 0:
                    end_indices = df[df.apply(lambda row: row.astype(str).str.contains('报关单统一编号').any(), axis=1)].index
                # 如果找到了“汇总”行和“报关单草稿”行
                if len(start_indices) > 0 and len(end_indices) > 0:
                    start_index = start_indices[0] + 1
                    end_index = end_indices[0]
                    # 只保留“商品序号”行到“报关单草稿”行（不包含）
                    df_filtered = df.iloc[start_index:end_index]
                    # 假设第一行是列名，将其设置为列名
                    if df_filtered.iloc[0].isnull().sum() == 0:
                        df_filtered.columns = df_filtered.iloc[0]
                        df_filtered = df_filtered[1:]
                    else:
                        st.warning(f"文件 {original_name} 的第一行没有有效的列名。")
                        continue
                    
                    # 只保留“商品序号”，“申报数量”和“申报总价”这三列
                    required_columns = ["备案序号", "申报数量", "申报总价"]
                    if all(col in df_filtered.columns for col in required_columns):
                        df_filtered = df_filtered[required_columns]
                    else:
                        st.warning(f"文件 {original_name} 中缺少必需的列。")
                        continue

                    # 转换申报数量和申报总价为数字类型
                    df_filtered["备案序号"] = pd.to_numeric(df_filtered["备案序号"], errors='coerce')
                    df_filtered["申报数量"] = pd.to_numeric(df_filtered["申报数量"], errors='coerce')
                    df_filtered["申报总价"] = pd.to_numeric(df_filtered["申报总价"], errors='coerce')
                    # 将处理后的数据添加到总数据列表中
                    all_data.append(df_filtered)
                else:
                    no_table_files.append(original_name)
        except Exception as e:
            st.warning(f"处理文件 {original_name} 时出错: {e}")
            no_table_files.append(original_name)

    if all_data:
        # 将所有数据合并为一个 DataFrame
        combined_df = pd.concat(all_data, ignore_index=True)
        # 计算累计申报数量和累计申报总价，并删除相同备案序号的行
        combined_df["累计申报数量"] = combined_df.groupby("备案序号")["申报数量"].transform('sum')
        combined_df["累计申报总价"] = combined_df.groupby("备案序号")["申报总价"].transform('sum')
        #...
        # 进一步处理 DataFrame 和返回结果
        return combined_df, no_table_files
    else:
        st.warning("没有找到有效的表格数据。")
        return None, no_table_files



