# -K3Cloud-
K3 Cloud是一款开放的ERP云平台.
一，产品的安装：
   1，SQL Server xxx
   2，K3Cloud产品安装包。(silverlight插件)
二，K3Cloud的开发前概述
   k3cloud支持两种语言以插件干预业务单据，实现业务逻辑的处理。(即C#(.net) , Python2.x)
   K3Cloud分层架构（ Web - APP - DataBase）
   K3Cloud插件干预不同的层(Web：表单插件、列表插件、表单构建插件； APP：服务插件)
三，业务单据对象的开发
    实现步骤：
   1，业务对象建模设计
   2，配置元素对象逻辑(关联配置、值更新、实体服务规则、单据转换、唯一检验等)
   3，插件干预业务对象逻辑
   4，发布并部署
==================================================================================
四，思践行
    《申请调拨单》
    4.1 核心节点一：
        1，判断【单据体(以list).需求量】字段，当为0或null时，实现【保存】删除单据体当前记录行。
        2，如下插件实现：
           思路：继承AbstractBillPlugIn，引用Kingdee.BOS,Kingdee.BOS.Core,Kingdee.BOS.DataEntity组件，添加Kingdee.BOS.Core.Bill.PlugIn，Kingdee.BOS.Orm.DataEntity，Kingdee.BOS.Core.Metadata.EntityElement实体对象，重载BarItemClick方法；
           插件：
           
                 public override void BarItemClick(Kingdee.BOS.Core.DynamicForm.PlugIn.Args.BarItemClickEventArgs e)
                  {
                      base.BarItemClick(e);
                      Entity entitys = this.View.BillBusinessInfo.GetEntity("FEntity");//单据分录实体元数据
                      DynamicObjectCollection cons = entitys.DynamicProperty.GetValue(this.Model.DataObject) as DynamicObjectCollection;
                      
                      if (e.BarItemKey == "tbSave" || e.BarItemKey == "tbSplitSave") //保存唯一标识
                      { 
                          //注解： i--遍历删除记录行，而非 i++处理删除记录行，是为了避免DeleteEntryRow执行导致Cons--，循环条件不满足，终止了循                                     环，造成不能一次性完成【保存】删除记录行。
                          for (int i = cons.Count - 1; i >= 0; i--) 
                          {
                              if (Convert.ToInt32(cons[i]["F_CMK_Qty3"]) == 0) 
                              {
                                this.View.Model.DeleteEntryRow("FEntity", i); 
                              }
                          }
                      }
                   }
                   
           
       4.2 核心节点二：
           1，判断【单据体.物料编码】，等于当前选中的单据头.仓库所在《即时库存(以list)》中该库存的物料编码一样时。则取当前《即时库存(list)》记录中的库存量字段值，并赋值给当前的《申请调拨单》的单据体(list)记录中。
           2，如下插件实现：
              思路：继承AbstractBillPlugIn，引用Kingdee.BOS,Kingdee.BOS.Core,Kingdee.BOS.DataEntity,Kingdee.BOS.ServiceHelper,添加Kingdee.BOS.Core.Bill.PlugIn,Kingdee.BOS.Orm.DataEntity,Kingdee.BOS.Core.Metadata.EntityElement,Kingdee.BOS.ServiceHelper,Kingdee.BOS.Core.Metadata实体对象，重载ButtonClick方法。
              插件：
                     
                  public override void ButtonClick(Kingdee.BOS.Core.DynamicForm.PlugIn.Args.ButtonClickEventArgs e)
                  {
                      base.ButtonClick(e);
                      List<SelectorItemInfo> selectItems = new List<SelectorItemInfo>(); //即时库存的字段片段列表
                      selectItems.Add(new SelectorItemInfo("FStockId"));
                      selectItems.Add(new SelectorItemInfo("FStockName"));
                      selectItems.Add(new SelectorItemInfo("FQty"));
                      selectItems.Add(new SelectorItemInfo("FMaterialId"));
                      selectItems.Add(new SelectorItemInfo("FMaterialName"));
                      selectItems.Add(new SelectorItemInfo("F_CMK_Text"));
                      OQLFilter filters = new OQLFilter(); //快捷过滤对象
                      filters.Add(new OQLFilterHeadEntityItem { FilterString = string.Format("FStockOrgId = {0}", this.Model.Context.CurrentOrganizationInfo.ID) }); 
                      //即时库存列表的集合数据
                      DynamicObject[] newLibraryDataModels = BusinessDataServiceHelper.Load(this.Context, "STK_Inventory", selectItems, filters);
                      Entity entitys = this.View.BillBusinessInfo.GetEntity("FEntity");//申请调拨单单据体的元数据模型
                      DynamicObjectCollection cons = entitys.DynamicProperty.GetValue(this.Model.DataObject) as DynamicObjectCollection;
                      DynamicObject dyObjects1 = this.View.Model.GetValue("F_CMK_WareHouseOne") as DynamicObject;//当前仓库列表
                      DynamicObject dyObjects2 = this.View.Model.GetValue("F_CMK_WareHouseTwo") as DynamicObject;
                      DynamicObject dyObjects3 = this.View.Model.GetValue("F_CMK_WareHouseThree") as DynamicObject;
                      String LibraryNumber01 = Convert.ToString(dyObjects1["Number"]); //申请调拨单的仓库一编码(此方式，要BOS上添加字段引用)
                      String LibraryNumber02 = Convert.ToString(dyObjects2["Number"]); //申请调拨单的仓库二编码  
                      String LibraryNumber03 = Convert.ToString(dyObjects3["Number"]); //申请调拨单的仓库三编码           
                      if (e.Key.ToUpper() == "F_CMK_BUTTON")
                      {
                         for (int i = 0; i < cons.Count; i++)
                         {
                             Decimal sum01 = 0; // 用于累加即时库存中多行同一物料的
                             Decimal sum02 = 0;
                             Decimal sum03 = 0;   
                             for (int j = 0; j < newLibraryDataModels.Count(); j++)
                             {
                                DynamicObject libraryDatas = newLibraryDataModels[j]["StockID"] as DynamicObject;
                                String LibraryNumber = Convert.ToString(libraryDatas["Number"]); //即时库存的仓库编码
                                DynamicObject materialDatas = cons[i]["F_CMK_Base3"] as DynamicObject;
                                String MaterialNumber = Convert.ToString(materialDatas["Number"]); //申请调拨单的物料编码
                                DynamicObject newLibaryDatas = newLibraryDataModels[j]["MaterialID"] as DynamicObject;
                                String MaterialNumber01 = Convert.ToString(newLibaryDatas["Number"]); //即时库存的物料编码
                                if(LibraryNumber01.Equals(LibraryNumber) && MaterialNumber.Equals(MaterialNumber01))
                                {
                                   sum01 += Convert.ToDecimal(newLibraryDataModels[j]["FQty"]);
                                   this.Model.SetValue("F_CMK_Qty", sum01, i);
                                }
                                if (LibraryNumber02.Equals(LibraryNumber) && MaterialNumber.Equals(MaterialNumber01))
                                {
                                   sum02 += Convert.ToDecimal(newLibraryDataModels[j]["FQty"]);
                                   this.Model.SetValue("F_CMK_Qty1", sum02, i);
                                }
                                if (LibraryNumber03.Equals(LibraryNumber) && MaterialNumber.Equals(MaterialNumber01))
                                {
                                   sum03 += Convert.ToDecimal(newLibraryDataModels[j]["FQty"]);
                                   this.Model.SetValue("F_CMK_Qty2", sum03, i);
                                }   
                           }    
                         }
                      }
                     this.View.UpdateView("FEntity");
                   }
              
