# flora-dp-federated-ColO-RAN
> 用於網路切片的差分隱私聯邦強化學習

[](https://www.google.com/search?q=https://www.gnu.org/licenses/agpl-3.0)

## 專案摘要

本專案在一個模擬的5G網路切片環境中，實作並評估了一個結合差分隱私（DP）的聯邦強化學習（FRL）框架。其目標是訓練一個能動態平衡增強型移動寬頻（eMBB）和超可靠低延遲通訊（URLLC）服務效能的智慧代理。

專案採用 PyTorch 和 Opacus（用於差分隱私）建構，並引入了創新的**虛擬客戶端生成**技術，以解決聯邦學習中客戶端數量不足的問題，同時增強隱私保護。

### 核心功能

  * **聯邦演算法**: 支援 `FedAvg`, `FedProx`, 和 `ClusteredFL`，並與集中式、孤立式訓練進行比較。
  * **差分隱私**: 使用 Opacus 函式庫為本地訓練提供強大的隱私保護，並包含智慧化的隱私預算管理與重設機制。
  * **虛擬客戶端生成**: 透過時間序列分割與特徵增強，從真實數據中生成統計特性相似但內容獨立的虛擬客戶端，以提升訓練穩定性。
  * **強化學習環境**: 一個為網路切片客製化的 DQN 代理與環境，其獎勵函數旨在平衡吞吐量與延遲。
  * **系統模擬**: 包含對客戶端掉線（dropout）和延遲（straggler）等真實世界場景的模擬。

## 數據集生成 (`kpi_traces_final_robust0.parquet`)

用於本專案訓練的 `kpi_traces_final_robust0.parquet` 檔案是透過一系列數據處理步驟，從一個公開的原始數據集生成的。這確保了數據的一致性、完整性與可用性。

### 1\. 原始數據來源

  * **數據庫**: `wineslab/colosseum-oran-coloran-dataset`。
  * **內容**: 該數據集來自 **Colosseum** 無線網路模擬器，模擬了一個多基站（multi-cell）的 O-RAN 環境。它包含了在不同排程策略（`sched*`）、訓練配置（`tr*`）和實驗運行（`exp*`）下，收集到的基站級（BS-level）和切片級（Slice-level）的效能指標（KPIs）。

### 2\. 數據處理流程

提供的程式碼（`Cell 3` 和 `Cell 4`）執行了一個兩階段的數據處理流程，將數百個原始 CSV 檔案轉換為單一的 Parquet 檔案。

  * **階段一：數據合併與豐富化**

    1.  **檔案探索**: 程式碼首先遞歸地查找所有基站級 (`bs*.csv`) 和切片級 (`*_metrics.csv`) 的 CSV 檔案。
    2.  **元數據提取**: 從每個檔案的路徑中解析出實驗元數據（如排程策略、基站ID、實驗ID），並將這些元數據作為新的欄位添加到數據中。
    3.  **數據清洗**: 對數據進行標準化，包括重新命名欄位（例如，`tx_brate downlink [Mbps]` 改為 `Throughput_DL_Mbps`）、移除無用欄位。
    4.  **中繼儲存**: 將處理後的基站數據和切片數據分別合併並儲存為兩個獨立的、高效的 Parquet 檔案（`kpi_traces_bs_intermediate0.parquet` 和 `kpi_traces_slice_intermediate0.parquet`）。

  * **階段二：時間感知合併**

    1.  **類型轉換與去重**: 載入中繼檔案，將時間戳（`timestamp`）轉換為標準的 `datetime` 格式，並移除重複的數據行以確保數據唯一性。
    2.  **時間序列合併**: 此流程最關鍵的一步是使用 `pandas.merge_asof` 進行時間感知合併。
          * **`direction='backward'`**: 該參數確保了因果關係的正確性。它將每個切片的數據點與其**之前最近**的基站數據點進行匹配。這避免了使用未來的基站狀態來描述當前的切片狀態，從而防止了數據洩漏。
          * **`tolerance='150ms'`**: 設定了一個 150 毫秒的時間容忍窗口來進行匹配。

### 3\. 最終輸出

  * 經過上述處理，最終生成了 `kpi_traces_final_robust0.parquet` 檔案。該檔案包含了一個統一的、按時間和實驗條件對齊的視圖，整合了所有基站和切片的效能指標，可以直接用於後續的強化學習模型訓練。


## 執行指南

本專案建議在 **Google Colab** 環境中執行以利用其免費 GPU 資源。

1.  **取得檔案**: 從本儲存庫下載以下兩個檔案：

      * `0701_FLORA_DP_client_15_v2_1 (1).ipynb`
      * `kpi_traces_final_robust0.parquet`

2.  **設定 Colab 環境**:

      * 前往 [Google Colab](https://colab.research.google.com/)，上傳 `.ipynb` 檔案。
      * 在選單中選擇 `Runtime > Change runtime type`，並將硬體加速器設為 **GPU**。

3.  **上傳數據與路徑設定**:

      * **建議**: 掛載您的 Google Drive，並在其中建立一個資料夾（例如 `FRL_Slicing_Sim`）。將 `kpi_traces_final_robust0.parquet` 檔案上傳至此。
      * **確認路徑**: 確保 `Cell 8B` 中的 `DATA_PATH` 變數指向正確的檔案路徑。

4.  **執行 Notebook**:

      * 依序執行 **`Cell 1`** 到 **`Cell 7`** 以定義所有必要的類別與函式。
      * 執行 **`Cell 8B`** 來設定實驗路徑。
      * **執行實驗**: 依序執行 **`Cell 8C`**, **`Cell 8D`**, 和 **`Cell 8E`** 來分別運行 `ClusteredFL`, `FedProx`, 和 `FedAvg` 的完整實驗。**此過程將花費大量時間**。
      * **檢視結果**: 待上述實驗完成後，執行 **`Cell 9`** 以生成視覺化圖表並分析結果。


## 授權條款

本專案採用 **GNU Affero General Public License v3.0 (AGPL-3.0)** 授權。

-----

# Differentially Private Federated Reinforcement Learning for Network Slicing

[](https://www.google.com/search?q=https://www.gnu.org/licenses/agpl-3.0)

## Project Summary

This project implements and evaluates a Federated Reinforcement Learning (FRL) framework combined with Differential Privacy (DP) in a simulated 5G network slicing environment. Its goal is to train an intelligent agent capable of dynamically balancing the performance of enhanced Mobile Broadband (eMBB) and Ultra-Reliable Low-Latency Communication (URLLC) services.

Built with PyTorch and Opacus (for Differential Privacy), the project introduces an innovative **Virtual Client Generation** technique to address the challenge of insufficient client availability in federated learning while enhancing privacy guarantees.

### Core Features

  * **Federated Algorithms**: Supports `FedAvg`, `FedProx`, and `ClusteredFL`, benchmarked against centralized and isolated training baselines.
  * **Differential Privacy**: Leverages the Opacus library to provide strong privacy guarantees for local training, featuring intelligent privacy budget management and reset mechanisms.
  * **Virtual Client Generation**: Creates statistically similar yet independent virtual clients from real data through time-series splitting and feature augmentation to improve training stability.
  * **Reinforcement Learning Environment**: A custom DQN agent and environment tailored for network slicing, with a reward function designed to balance throughput and latency.
  * **System Simulation**: Includes simulation of real-world scenarios such as client dropouts and stragglers.


## Dataset Generation (`kpi_traces_final_robust0.parquet`)

The `kpi_traces_final_robust0.parquet` file used for training in this project is generated from a public raw dataset through a series of data processing steps. This ensures data consistency, integrity, and usability.

### 1\. Raw Data Source

  * **Database**: `wineslab/colosseum-oran-coloran-dataset`.
  * **Content**: This dataset originates from the **Colosseum** wireless network emulator, simulating a multi-cell O-RAN environment. It contains base station (BS-level) and slice-level Key Performance Indicators (KPIs) collected under various scheduling policies (`sched*`), training configurations (`tr*`), and experiment runs (`exp*`).

### 2\. Data Processing Pipeline

The provided code (`Cell 3` and `Cell 4`) executes a two-stage data processing pipeline to convert hundreds of raw CSV files into a single Parquet file.

  * **Stage 1: Data Consolidation and Enrichment**

    1.  **File Discovery**: The code first recursively discovers all BS-level (`bs*.csv`) and slice-level (`*_metrics.csv`) CSV files.
    2.  **Metadata Extraction**: It parses experimental metadata (e.g., scheduling policy, BS ID, experiment ID) from each file's path and adds this information as new columns to the data.
    3.  **Data Cleaning**: The data is standardized, including renaming columns (e.g., `tx_brate downlink [Mbps]` to `Throughput_DL_Mbps`) and removing unused columns.
    4.  **Intermediate Storage**: The processed base station and slice data are then consolidated and saved into two separate, efficient Parquet files (`kpi_traces_bs_intermediate0.parquet` and `kpi_traces_slice_intermediate0.parquet`).

  * **Stage 2: Time-Aware Merging**

    1.  **Type Conversion & Deduplication**: The intermediate files are loaded, timestamps are converted to a standard `datetime` format, and duplicate rows are removed to ensure data uniqueness.
    2.  **Time-Series Merge**: The most critical step in this pipeline is the time-aware merge using `pandas.merge_asof`.
          * **`direction='backward'`**: This parameter ensures causal correctness. It matches each slice's data point with the **most recent preceding** base station data point. This avoids data leakage by preventing the use of future base station states to describe the current slice state.
          * **`tolerance='150ms'`**: A 150ms time tolerance window is set for matching records.

### 3\. Final Output

  * After this process, the final `kpi_traces_final_robust0.parquet` file is generated. This file provides a unified, time-aligned view of all base station and slice KPIs, ready to be used directly for training the reinforcement learning model.

## Execution Guide

This project is best run in **Google Colab** to utilize its free GPU resources.

1.  **Obtain Files**: Download the following two files from this repository:

      * `0701_FLORA_DP_client_15_v2_1 (1).ipynb`
      * `kpi_traces_final_robust0.parquet`

2.  **Set Up Colab Environment**:

      * Go to [Google Colab](https://colab.research.google.com/) and upload the `.ipynb` file.
      * From the menu, select `Runtime > Change runtime type` and set the hardware accelerator to **GPU**.

3.  **Upload Data & Configure Path**:

      * **Recommended**: Mount your Google Drive, create a folder (e.g., `FRL_Slicing_Sim`), and upload the `kpi_traces_final_robust0.parquet` file there.
      * **Verify Path**: Ensure the `DATA_PATH` variable in `Cell 8B` points to the correct file location.

4.  **Run the Notebook**:

      * Sequentially execute **`Cell 1`** through **`Cell 7`** to define all necessary classes and functions.
      * Execute **`Cell 8B`** to configure experiment paths.
      * **Run Experiments**: Execute **`Cell 8C`**, **`Cell 8D`**, and **`Cell 8E`** to run the full experiments for `ClusteredFL`, `FedProx`, and `FedAvg`, respectively. **This process is time-consuming**.
      * **View Results**: Once the experiments are complete, run **`Cell 9`** to generate visualizations and analyze the results.

## License

This project is licensed under the **GNU Affero General Public License v3.0 (AGPL-3.0)**.
