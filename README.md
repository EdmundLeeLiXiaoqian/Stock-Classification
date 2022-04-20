# Stock-Classification
该代码是基于AP近邻传播聚类算法对股票进行分类，具体代码在ipynb中  
本实验需要采集国内股票的基础信息、交易数据和财务指标数据，基于这两种数据做聚类、预测以及实验结果的分析。股票交易数据的采集主要通过Python API采集TuShare财经数据接口包的数据。TuShare提供了开源的数据API，该API是通过Python实现的。为了能够提供给用户方便快捷、快速易用的服务，使得获取到的数据具有多样性和可分析性，TuShare的API对网上的海量数据从采集到清洗再到加工实现了一体化服务。金融量化分析的研究和实践的过程中Python pandas 工具包扮演了重要的角色，因此Tushare返回的绝大部分的数据格式都是pandas DataFrame类型的，易于数据的持久化，易于pandas/NumPy进行数据的分析，易于Matplotlib进行分析结果的可视化。此外，TuShare提供的SQL，比如 Mysql以及NoSQL，比如 MongoDB，用户可以使用Tushare的API直接进行数据的持久化，将数据全部保存到本地后再进行处理分析。  
从TuShare财经数据API拿到的数据是以Pandas DataFrame的形式封装的，拿到数据后为了后续的实验，需要将数据进行数据的持久化，数据的预处理工作基于序列化数据，Python及DataFrame提供了封装好的Api支持文件的写入，因此我们将数据存到excel表格中。在经过数据的剔除、数据归一化、数据稀疏性的均值填充后，可以拿到时间序列矩阵。  
实验过程中所需数据分成股票财务指标数据和股票日线行情数据，股票财务指标数据包括股票代码（ts_code），每股收益（eps），净额现金流（ocfps）。股票的日线数据包括股票代码（ts_code），交易日期（trade_date），当日收盘价（close），换手率（turnover_rate），自由流通股的换手率（turnover_rate_f），量比（volume_ratio），市盈率（pe），TTM市盈率（pe_ttm），市净率（pb），市销率（ps），TTM市销率（ps_ttm），总股本（total_share），流通股本（float_share），自由流通股本（free_share），总市值（total_mv），流通市值（circ_mv）等。  
实验过程中用到的数据基于TuShare Pro的API并不能直接取到，具体步骤如下：  
（1）安装tushare到本地，本地需安装python，pandas，anaconda  
（2）获取账户Token后通过方法ts.set_token设置账户Token  
（3）获取股票基本信息，通过执行pro.stock_basic方法得到股票List  
（4）遍历股票List获取每只股票的日线数据  
（5）梳理数据为标准的输入矩阵，行表示每支股票的特征，列表示每个时间点的所有股票的特征，同时每行的顺序表示特定股票的映射已持久化。  
本实验我选择上证50指数的成分股5年的日线数据作为研究对象。  
由于tushare模块已经将股票数据进行了基本的规整，此处我们只需要将数据处理成我们项目所需要的样子即可。  
此处对股票数据的规整包括有几个  
(1)计算需要聚类的数据，此处我用收盘价减去开盘价作分析，即一天的涨跌幅度。  
(2)由于使用get_batch_k_df() 函数获取的批量股票数据都是将多个股票数据在纵向上合并而来，故而此处我们要将各种不同股票的涨跌幅度放在DataFrame的列上，以股票代码为列名。  
(3)在pd.merge()过程中，由于有的股票在某些交易日停牌，所以没有数据，这几个交易日就被删掉（因为后面的聚类算法中不允许存在NaN)，所以相当于要选择所有股票都有交易数据的日期，这个选择相当于取股票数据的交集，最终得到很少一部分数据，数据量太少时，得到的聚类结果也没有太多说服力。故而我的解决方法是，删除一些交易日明显很少的股票，不对其进行pd.merge()。最终得到882个交易日的有效数据，选取了49只股票，舍弃了1只停牌日太多的股票。  
(4)进行数据归一化：  
在深度学习领域，许多模型都需要进行归一化处理。存在模型进行了特征的各个维度的不均匀伸缩后，收敛后的最优解和原来的不一样。使用该类模型的时候需要进行数据的标准化或者归一化，倘若数据集中的各个维度的数据分布范围比较集中，可以不进行数据归一化，否则必需归一化，以免模型参数被分布范围较大或较小的数据支配。实际应用中也存在模型的特征标准化后收敛后的最优解和原来一样的，但是如果不标准化可能收敛得很慢甚至不收敛，因此标准化很必要。  
数据的标准化（Normalization）的本质是将实验过程中数据的非数值的含义转换程可以量化的数值型数据，对外的表现为实验数据的规范到一个固定的自定义区间内。机器学习领域在评估或比较模型的性能时用到数据的标准化，将实验过程中数据的非数值的含义转换程可以量化的数值型数据，便于不同含义或量化的特征能够进行比较计算。数据的归一化是数据标准化的一种，归一化的方法多种多样，股票的交易的数据可以以增长比例作为归一化的方法，即   
z_ij=(x_(i(j+1))-x_ij )/x_ij （ 3 1 ）  
其中:i ={1,2,…,m};□( ) j={1,2,…n-1};z∈[-1,1] 。  
数据归一化之后有两个比较明显的优点：提升模型的收敛速度及提升模型的精度。  
通过不同的方式描述股票之间的相似性，之后通过拼接不同的聚类算法对股票进行类别的划分探索对于股票时序数据而言，不同方法相似性之间表达的优劣，以及不同聚类算法之间的差异，挑选出最合适的聚类算法用于股票分类，增强对股票分类的准确性。  
最后得到：  
Cluster: 0----> stocks: 浦发银行,华夏银行,民生银行,招商银行,中国联通,大秦铁路,兴业银行,北京银行,农业银行,交通银行,工商银行,中国建筑,中国南车,光大银行  
Cluster: 1----> stocks: 上港集团,中信证券,上汽集团,百视通,海通证券,招商证券,华泰证券,方正证券,中国重工  
Cluster: 2----> stocks: 保利地产,金地集团,海螺水泥  
Cluster: 3----> stocks: 包钢股份,包钢稀土,厦门钨业  
Cluster: 4----> stocks: 江西铜业,紫金矿业  
Cluster: 5----> stocks: 国电南瑞  
Cluster: 6----> stocks: 中金黄金,山东黄金  
Cluster: 7----> stocks: 白云山,康美药业  
Cluster: 8----> stocks: 三一重工,贵州茅台,伊利股份  
Cluster: 9----> stocks: 中国石化,广汇能源,中国神华,中国化学,潞安环能,中国石油  
Cluster: 10----> stocks: 中国平安,新华保险,中国太保,中国人寿  
