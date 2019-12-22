---
layout:     post
title:      Oracle管道函数
subtitle:   管道函数demo
date:       2019-08-11
author:     Yuanye.Wang
header-img: img/post-bg01-1115.jpg
catalog: false
tags:
    - Oracle
---
> 1.声明
```sql
CREATE OR REPLACE PACKAGE hcrm_sales_peformance_pub AS
     TYPE type_sales_peformance_rec IS RECORD(
        the_year           NUMBER,
        sale_area_id       NUMBER,
        sale_area_name     VARCHAR2(200),
        sale_team_id       NUMBER,
        sale_team_name     VARCHAR2(200),
        sale_employee_id   NUMBER,
        sale_employee_code   VARCHAR2(200),
        sale_employee_name VARCHAR2(200),
        hot_money          NUMBER,
        pro_money          NUMBER,
        win_money          NUMBER,
        sal_clo            NUMBER,
        win_sal_clo        NUMBER);
    
     TYPE type_sales_peformance_tbl IS TABLE OF type_sales_peformance_rec;
    
     FUNCTION report(p_year        number,
                        p_org_id      number,
                        p_lang        varchar2) RETURN type_sales_peformance_tbl
     PIPELINED;

END hcrm_sales_peformance_pub;
```
> 2.包体
```sql
CREATE OR REPLACE PACKAGE BODY hcrm_sales_peformance_pub AS


  --获取报表明细
  FUNCTION get_report_detail(p_year        number,
                             p_org_id      number,
                             p_lang        varchar2) RETURN type_sales_peformance_tbl IS
     l_org_type              varchar2(100);
     l_organization_name     varchar2(300);
     l_seq  number:=1;
     l_sales_peformance_rec  type_sales_peformance_rec;
     l_sales_peformance_tbl  type_sales_peformance_tbl:=type_sales_peformance_tbl();
  BEGIN
    
    FOR l_rec IN (select hop.sale_area_id,
                        (select hob.organization_name
                           from hcrm.hcrm_organization_b hob
                          where hob.organization_id = hop.sale_area_id) sale_area_name,
                        hop.sale_team_id,
                        (select hob.organization_name
                           from hcrm.hcrm_organization_b hob
                          where hob.organization_id = hop.sale_team_id) sale_team_name,
                        hop.sale_employee_id,
                        (select bev.employee_code
                           from hcrm.hcrm_base_employee_view bev
                          where bev.user_id = hop.sale_employee_id) sale_employee_code,
                        (select bev.name
                           from hcrm.hcrm_base_employee_view bev
                          where bev.user_id = hop.sale_employee_id) sale_employee_name,
                        hop.the_year,
                        sum(hop.hot_money*hop.exchange_rate) hot_money,
                        sum(hop.pro_money*hop.exchange_rate) pro_money,
                        sum(hop.win_money*hop.exchange_rate) win_money,
                        sum((hop.sal_money+hop.clo_money)*hop.exchange_rate) sal_clo,
                        sum((hop.win_money+hop.sal_money+hop.clo_money)*hop.exchange_rate) win_sal_clo
                  from (select ho.sale_area_id,
                               ho.sale_team_id,
                               ho.sale_employee_id,
                               ho.currency,
                               ho.opportunity_status,
                               ho.the_year,
                               nvl((select hber.exchange_rate
                                      from hcrm.hom_base_exchange_rate hber
                                     where hber.finnal_currency = 'HCRM_CNY'
                                       and hber.is_valid = 'Y'
                                       and hber.original_currency = ho.currency
                                       and rownum = 1),1) exchange_rate,
                               (case ho.opportunity_status when 'HCRM_HOT_CHANCE' then sum(nvl(ho.pre_money,0)) else 0 end) hot_money,   --热门
                               (case ho.opportunity_status when 'HCRM_PROBABLY_WIN' then sum(nvl(ho.pre_money,0)) else 0 end) pro_money, --赢面大
                               (case ho.opportunity_status when 'HCRM_WIN_CONFIRMED' then sum(nvl(ho.pre_money,0)) else 0 end) win_money,--赢单
                               (case ho.opportunity_status when 'HCRM_SALE_CLOSED' then sum(nvl(ho.pre_money,0)) else 0 end) sal_money,  --关闭
                               (case ho.opportunity_status when 'HCRM_CLOSED' then sum(nvl(ho.pre_money,0)) else 0 end) clo_money        --归档
                          from hcrm.hcrm_opportunity ho
                         where ho.opportunity_type = 'HCRM_OPPORTUNITY'
                           and ho.status = 'HCRM_ENABLE'
                           --这里要加限制，不然其他状态的商机会占一行
                           and ho.opportunity_status in ('HCRM_HOT_CHANCE','HCRM_PROBABLY_WIN','HCRM_WIN_CONFIRMED','HCRM_SALE_CLOSED','HCRM_CLOSED')
                           and ho.the_year = p_year
                           and exists (select 1
                                         from hcrm.hcrm_org_parent_index hopi
                                        where hopi.child_org_id = nvl(ho.sale_team_id,ho.sale_area_id)
                                          and hopi.organization_id = p_org_id )
                         group by ho.sale_area_id,
                                  ho.sale_team_id,
                                  ho.sale_employee_id,
                                  ho.currency,
                                  ho.opportunity_status,
                                  ho.the_year) hop
                    group by hop.sale_area_id,
                             hop.sale_team_id,
                             hop.sale_employee_id,
                             hop.the_year) 
      LOOP
            l_sales_peformance_rec.the_year           := l_rec.the_year;
            l_sales_peformance_rec.sale_area_id       := l_rec.sale_area_id ;  
            l_sales_peformance_rec.sale_area_name     := l_rec.sale_area_name;
            l_sales_peformance_rec.sale_team_id       := l_rec.sale_team_id;
            if l_rec.sale_team_id is not null then
              select ha.org_type,ha.organization_name
              into l_org_type,l_organization_name
              from HCRM_ORGANIZATION_B ha, HCRM_ORGANIZATION_B hb
               where ha.organization_id = hb.parent_org_id
               and hb.organization_id = l_rec.sale_team_id
               and rownum=1; 
               if l_org_type = 'HCRM_SALE_AREA' then
                l_sales_peformance_rec.sale_team_name     := l_rec.sale_team_name;
              else
                l_sales_peformance_rec.sale_team_name     := l_organization_name ||'-'|| l_rec.sale_team_name;
              end if;
            else
               l_sales_peformance_rec.sale_team_name     := l_rec.sale_team_name;
            end if;
            l_sales_peformance_rec.sale_employee_id   := l_rec.sale_employee_id;
            l_sales_peformance_rec.sale_employee_code := l_rec.sale_employee_code;
            l_sales_peformance_rec.sale_employee_name := l_rec.sale_employee_name;
            l_sales_peformance_rec.hot_money          := l_rec.hot_money;
            l_sales_peformance_rec.pro_money          := l_rec.pro_money;
            l_sales_peformance_rec.win_money          := l_rec.win_money;
            l_sales_peformance_rec.sal_clo            := l_rec.sal_clo;
            l_sales_peformance_rec.win_sal_clo        := l_rec.win_sal_clo;
            
            l_sales_peformance_tbl.extend;
            l_sales_peformance_tbl(l_seq) := l_sales_peformance_rec;
            l_seq := l_seq + 1;
      END LOOP;
      RETURN l_sales_peformance_tbl;
  END;
  
  --获取团队统计
  FUNCTION get_team_total(p_year        number,
                          p_org_id      number,
                          p_lang        varchar2,
                          p_flag        out number) RETURN type_sales_peformance_rec IS
  l_sales_peformance_rec  type_sales_peformance_rec;
  l_org_type              varchar2(100);
  l_organization_name     varchar2(300);
  BEGIN                           
    
     FOR l_rec IN( select  hop.sale_area_id,
                          (select hob.organization_name
                             from hcrm.hcrm_organization_b hob
                            where hob.organization_id = hop.sale_area_id) sale_area_name,
                          hop.sale_team_id,
                          (select hob.organization_name||'汇总'
                             from hcrm.hcrm_organization_b hob
                            where hob.organization_id = hop.sale_team_id) sale_team_name,
                          99999 as sale_employee_id,
                          99999 as sale_employee_code,
                          '-' sale_employee_name,
                          hop.the_year,
                          sum(hop.hot_money*hop.exchange_rate) hot_money,
                          sum(hop.pro_money*hop.exchange_rate) pro_money,
                          sum(hop.win_money*hop.exchange_rate) win_money,
                          sum((hop.sal_money+hop.clo_money)*hop.exchange_rate) sal_clo,
                          sum((hop.win_money+hop.sal_money+hop.clo_money)*hop.exchange_rate)  win_sal_clo
                    from (select ho.sale_area_id,
                                 p_org_id as sale_team_id,
                                 ho.currency,
                                 ho.opportunity_status,
                                 ho.the_year,
                                 nvl((select hber.exchange_rate
                                        from hcrm.hom_base_exchange_rate hber
                                       where hber.finnal_currency = 'HCRM_CNY'
                                         and hber.is_valid = 'Y'
                                         and hber.original_currency = ho.currency
                                         and rownum = 1),1) exchange_rate,
                                 (case ho.opportunity_status when 'HCRM_HOT_CHANCE' then sum(nvl(ho.pre_money,0)) else 0 end) hot_money,   --热门
                                 (case ho.opportunity_status when 'HCRM_PROBABLY_WIN' then sum(nvl(ho.pre_money,0)) else 0 end) pro_money, --赢面大
                                 (case ho.opportunity_status when 'HCRM_WIN_CONFIRMED' then sum(nvl(ho.pre_money,0)) else 0 end) win_money,--赢单
                                 (case ho.opportunity_status when 'HCRM_SALE_CLOSED' then sum(nvl(ho.pre_money,0)) else 0 end) sal_money,  --关闭
                                 (case ho.opportunity_status when 'HCRM_CLOSED' then sum(nvl(ho.pre_money,0)) else 0 end) clo_money        --归档
                            from hcrm.hcrm_opportunity ho
                           where ho.opportunity_type = 'HCRM_OPPORTUNITY'
                             and ho.status = 'HCRM_ENABLE'
                             --这里要加限制，不然其他状态的商机会占一行
                             and ho.opportunity_status in ('HCRM_HOT_CHANCE','HCRM_PROBABLY_WIN','HCRM_WIN_CONFIRMED','HCRM_SALE_CLOSED','HCRM_CLOSED')
                             and exists (select 1
                                           from hcrm.hcrm_org_parent_index hopi
                                          where hopi.child_org_id = nvl(ho.sale_team_id,ho.sale_area_id)
                                            and hopi.organization_id = p_org_id )
                             and ho.the_year = p_year
                           group by ho.sale_area_id,
                                    ho.sale_team_id,
                                    ho.currency,
                                    ho.opportunity_status,
                                    ho.the_year) hop
                      group by hop.sale_area_id,
                               hop.sale_team_id,
                               hop.the_year) 
      LOOP
            l_sales_peformance_rec.the_year           := l_rec.the_year;
            l_sales_peformance_rec.sale_area_id       := l_rec.sale_area_id ;  
            l_sales_peformance_rec.sale_area_name     := l_rec.sale_area_name;
            l_sales_peformance_rec.sale_team_id       := l_rec.sale_team_id;
            if l_rec.sale_team_id is not null then
              select ha.org_type,ha.organization_name
              into l_org_type,l_organization_name
              from HCRM_ORGANIZATION_B ha, HCRM_ORGANIZATION_B hb
               where ha.organization_id = hb.parent_org_id
               and hb.organization_id = l_rec.sale_team_id
               and rownum=1; 
               if l_org_type = 'HCRM_SALE_AREA' then
                l_sales_peformance_rec.sale_team_name     := l_rec.sale_team_name;
              else
                l_sales_peformance_rec.sale_team_name     := l_organization_name ||'-'|| l_rec.sale_team_name;
              end if;
            else
               l_sales_peformance_rec.sale_team_name     := l_rec.sale_team_name;
            end if;
            l_sales_peformance_rec.sale_employee_id   := l_rec.sale_employee_id;
            l_sales_peformance_rec.sale_employee_code := l_rec.sale_employee_code;
            l_sales_peformance_rec.sale_employee_name := l_rec.sale_employee_name;
            l_sales_peformance_rec.hot_money          := l_rec.hot_money;
            l_sales_peformance_rec.pro_money          := l_rec.pro_money;
            l_sales_peformance_rec.win_money          := l_rec.win_money;
            l_sales_peformance_rec.sal_clo            := l_rec.sal_clo;
            l_sales_peformance_rec.win_sal_clo        := l_rec.win_sal_clo;
          
            p_flag := 1;
            RETURN l_sales_peformance_rec;
           
      END LOOP;
      p_flag := 0;
      RETURN NULL;
  END;
  
  --获取大区统计
  FUNCTION get_area_total(p_year        number,
                          p_org_id      number,
                          p_lang        varchar2,
                          p_flag        out number) RETURN type_sales_peformance_rec IS
  l_sales_peformance_rec  type_sales_peformance_rec;
  BEGIN                           
    
     FOR l_rec IN( select hop.sale_area_id,
                          (select hob.organization_name||'汇总'
                             from hcrm.hcrm_organization_b hob
                            where hob.organization_id = hop.sale_area_id) sale_area_name,
                          99999 as sale_team_id,
                          '-' sale_team_name,
                          99999 as sale_employee_id,
                          99999 as sale_employee_code,
                          '-' sale_employee_name,
                          hop.the_year,
                          sum(hop.hot_money*hop.exchange_rate) hot_money,
                          sum(hop.pro_money*hop.exchange_rate) pro_money,
                          sum(hop.win_money*hop.exchange_rate) win_money,
                          sum((hop.sal_money+hop.clo_money)*hop.exchange_rate) sal_clo,
                          sum((hop.win_money+hop.sal_money+hop.clo_money)*hop.exchange_rate) win_sal_clo
                    from (select p_org_id as sale_area_id,
                                 ho.currency,
                                 ho.opportunity_status,
                                 ho.the_year,
                                 nvl((select hber.exchange_rate
                                        from hcrm.hom_base_exchange_rate hber
                                       where hber.finnal_currency = 'HCRM_CNY'
                                         and hber.is_valid = 'Y'
                                         and hber.original_currency = ho.currency
                                         and rownum = 1),1) exchange_rate,
                                 (case ho.opportunity_status when 'HCRM_HOT_CHANCE' then sum(nvl(ho.pre_money,0)) else 0 end) hot_money,   --热门
                                 (case ho.opportunity_status when 'HCRM_PROBABLY_WIN' then sum(nvl(ho.pre_money,0)) else 0 end) pro_money, --赢面大
                                 (case ho.opportunity_status when 'HCRM_WIN_CONFIRMED' then sum(nvl(ho.pre_money,0)) else 0 end) win_money,--赢单
                                 (case ho.opportunity_status when 'HCRM_SALE_CLOSED' then sum(nvl(ho.pre_money,0)) else 0 end) sal_money,  --关闭
                                 (case ho.opportunity_status when 'HCRM_CLOSED' then sum(nvl(ho.pre_money,0)) else 0 end) clo_money        --归档
                            from hcrm.hcrm_opportunity ho
                           where ho.opportunity_type = 'HCRM_OPPORTUNITY'
                             and ho.status = 'HCRM_ENABLE'
                             --这里要加限制，不然其他状态的商机会占一行
                             and ho.opportunity_status in ('HCRM_HOT_CHANCE','HCRM_PROBABLY_WIN','HCRM_WIN_CONFIRMED','HCRM_SALE_CLOSED','HCRM_CLOSED')
                             and exists (select 1
                                           from hcrm.hcrm_org_parent_index hopi
                                          where hopi.child_org_id = nvl(ho.sale_team_id,ho.sale_area_id)
                                            and hopi.organization_id = p_org_id )
                             and ho.the_year = p_year
                           group by ho.sale_area_id,
                                    ho.currency,
                                    ho.opportunity_status,
                                    ho.the_year) hop
                      group by hop.sale_area_id,
                               hop.the_year) 
      LOOP
            l_sales_peformance_rec.the_year           := l_rec.the_year;
            l_sales_peformance_rec.sale_area_id       := l_rec.sale_area_id ;  
            l_sales_peformance_rec.sale_area_name     := l_rec.sale_area_name;
            l_sales_peformance_rec.sale_team_id       := l_rec.sale_team_id;
            l_sales_peformance_rec.sale_team_name     := l_rec.sale_team_name;
            l_sales_peformance_rec.sale_employee_id   := l_rec.sale_employee_id;
            l_sales_peformance_rec.sale_employee_code := l_rec.sale_employee_code;
            l_sales_peformance_rec.sale_employee_name := l_rec.sale_employee_name;
            l_sales_peformance_rec.hot_money          := l_rec.hot_money;
            l_sales_peformance_rec.pro_money          := l_rec.pro_money;
            l_sales_peformance_rec.win_money          := l_rec.win_money;
            l_sales_peformance_rec.sal_clo            := l_rec.sal_clo;
            l_sales_peformance_rec.win_sal_clo        := l_rec.win_sal_clo;
          
            p_flag := 1;
            RETURN l_sales_peformance_rec;
           
      END LOOP;
      p_flag := 0;
      RETURN NULL;
  END;
  
  
  /*==================================================
  Function Name :
      report
  Description:
      This function is for:
           返回报表展示数据
  Argument:
      p_year               : 年份
      p_org_id             : 部门ID
      p_lang               : 语言
  ==================================================*/
  FUNCTION report(p_year        number,
                  p_org_id      number,
                  p_lang        varchar2) RETURN type_sales_peformance_tbl PIPELINED IS
     l_sales_peformance_rec  type_sales_peformance_rec;
     l_sales_peformance_tbl  type_sales_peformance_tbl;
     l_seq number;
     l_flag number;
  BEGIN
      l_sales_peformance_tbl := get_report_detail( p_year    => p_year,
                                                   p_org_id  => p_org_id,
                                                   p_lang    => p_lang);
      
      IF nvl(l_sales_peformance_tbl.count,0) = 0
        THEN
          RETURN;
      END IF;
      
       l_seq := l_sales_peformance_tbl.count;
      
      FOR l_child_unit_rec IN( select hob.organization_id,
                                      hob.org_type
                                 from hcrm.hcrm_org_parent_index hopi,
                                      hcrm.hcrm_organization_b hob
                                where hopi.organization_id = p_org_id
                                  and hopi.child_org_id = hob.organization_id
                                 group by hob.organization_id,hob.org_type)
      LOOP
          l_sales_peformance_rec := NULL;
          l_flag := 0;
          
          IF l_child_unit_rec.org_type = 'HCRM_SALE_AREA'
            THEN 
              l_sales_peformance_rec := get_area_total( p_year   => p_year,
                                                        p_org_id => l_child_unit_rec.organization_id,
                                                        p_lang   => p_lang,
                                                        p_flag   => l_flag);
          ELSIF l_child_unit_rec.org_type = 'HCRM_SALE_TEAM'
            THEN 
              l_sales_peformance_rec := get_team_total( p_year   => p_year,
                                                        p_org_id => l_child_unit_rec.organization_id,
                                                        p_lang   => p_lang,
                                                        p_flag   => l_flag);
             
          END IF;
          
          IF l_flag <> 0
                THEN
                   l_sales_peformance_tbl.extend();
                   l_seq := l_seq + 1;
                   l_sales_peformance_tbl(l_seq) := l_sales_peformance_rec;
          END IF;
          
      END LOOP;
      
      FOR i IN l_sales_peformance_tbl.first .. l_sales_peformance_tbl.last LOOP
        l_sales_peformance_rec := l_sales_peformance_tbl(i);
        PIPE ROW(l_sales_peformance_rec);
      END LOOP;
      
      RETURN;
  END;

END hcrm_sales_peformance_pub;

```