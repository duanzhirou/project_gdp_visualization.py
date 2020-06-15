# project_gdp_visualization.py
#coding:gbk
"""
综合项目:世行历史数据基本分类及其可视化
作者：段至柔
日期：6月15日
"""

import csv
import math
import pygal
import pygal_maps_world  # 导入需要使用的库


def read_csv_as_nested_dict(filename, keyfield, separator, quote):  # 读取原始csv文件的数据，格式为嵌套字典
    """
    输入参数:
      filename:csv文件名
      keyfield:键名
      separator:分隔符
      quote:引用符
    输出:
      读取csv文件数据，返回嵌套字典格式，其中外层字典的键对应参数keyfiled，内层字典对应每行在各列所对应的具体值
    """
    result = {}
    with open(filename, newline="") as csvfile:
        csvreader = csv.DictReader(csvfile, delimiter=separator, quotechar=quote)
        for row in csvreader:
            rowid = row[keyfield]
            result[rowid] = row

    return result


# pygal_countries = pygal.maps.world.COUNTRIES #读取pygal.maps.world中国家代码信息（为字典格式），其中键为pygal中各国代码，值为对应的具体国名(建议将其显示在屏幕上了解具体格式和数据内容）
# print(pygal_countries)


def reconcile_countries_by_name(plot_countries, gdp_countries):  # 返回在世行有GDP数据的绘图库国家代码字典，以及没有世行GDP数据的国家代码集合
    """
    输入参数:
    plot_countries: 绘图库国家代码数据，字典格式，其中键为绘图库国家代码，值为对应的具体国名
    gdp_countries:世行各国数据，嵌套字典格式，其中外部字典的键为世行国家代码，值为该国在世行文件中的行数据（字典格式)
    输出：
    返回元组格式，包括一个字典和一个集合。其中字典内容为在世行有GDP数据的绘图库国家信息（),
    集合内容为在世行无GDP数据的绘图库国家代码
    """
    
    plot_gdp_countries_yes={}        #世行有GDP数据的绘图库国家,记入字典  
    not_plot_gdp_countries=set()        #世行无GDP数据的绘图库国家，记入集合
    for m in gdp_countries.values():    #判断世行GDP是否有GDP数据       
        if any(list(m.values())[4:]):
            country=m['Country Name']      #如果有绘图库国家的代码，就在字典中加上            
            if country in plot_countries.values():
                key=list(plot_countries.keys())[(list(plot_countries.values()).index(country))]
                plot_gdp_countries_yes[key]=country       #键为绘图库各国家代码，值为对应的具体国名
        else:
            country=m['Country Name']
            if country in plot_countries.values():
                key=list(plot_countries.keys())[(list(plot_countries.values()).index(country))]
                not_plot_gdp_countries.add(key)          #没有GDP数据，加在集合上

    return tuple([plot_gdp_countries_yes, not_plot_gdp_countries])     #返回元组格式，包括一个字典和一个集合。其中字典内容为在世行有GDP数据的绘图库国家信息




def build_map_dict_by_name(gdpinfo, plot_countries, year):
    """
    输入参数:
    gdpinfo: gdp信息
	plot_countries: 绘图库国家代码数据，字典格式，其中键为绘图库国家代码，值为对应的具体国名
	year: 具体年份值
    输出：
    输出包含一个字典和二个集合的元组数据。其中字典数据为绘图库各国家代码及对应的在某具体年份GDP产值（键为绘图库中各国家代码，值为在具体年份（由year参数确定）所对应的世行GDP数据值。为
    后续显示方便，GDP结果需转换为以10为基数的对数格式，如GDP原始值为2500，则应为log2500，ps:利用math.log()完成)
    2个集合一个为在世行GDP数据中完全没有记录的绘图库国家代码，另一个集合为只是没有某特定年（由year参数确定）世行GDP数据的绘图库国家代码
   """
   
    a=read_csv_as_nested_dict(gdpinfo['gdpfile'], gdpinfo['country_name'], gdpinfo['separator'],gdpinfo['quote'])
    country_gdp_in_a_year={}            #可绘图的国家与完全不可绘图的国家，plot_countries 是reconcile_countries_by_name()函数的返回值   
    plot_countries_yes,not_plot_countries=reconcile_countries_by_name(plot_countries,a)       #没有某特定年世行GDP数据但可以绘图的绘图库国家代码
    plot_countries_no_gdp=set()     #世行GDP数据中完全没有记录的绘图库国家代码
    all_countries_gdp={}
    for n in a.values():
        i=dict(zip(n.keys(), n.values()))
        all_countries_gdp[i['Country Name']]=i
    for code, country in plot_countries_yes.items():
        if country in all_countries_gdp.keys():
            year_gdp=all_countries_gdp[country][str(year)]
            if year_gdp:
                country_gdp_in_a_year[code]=math.log10(float(year_gdp))    #利用math.log()将GDP结果转换为以10为基数的对数格式
            else:
                plot_countries_no_gdp.add(code)

    return tuple([country_gdp_in_a_year, plot_countries_no_gdp, not_plot_countries])#输出包含一个字典即绘图库各国家代码及对应的在某具体年份GDP产值和二个集合即在世行GDP数据中完全没有记录的绘图库国家代码，另一个集合为只是没有某特定年（由year参数确定）世行GDP数据的绘图库国家代码的元组数据


def render_world_map(gdpinfo, plot_countries, year, map_file):  # 将具体某年世界各国的GDP数据(包括缺少GDP数据以及只是在该年缺少GDP数据的国家)以地图形式可视化
    """
    Inputs:
    gdpinfo:gdp信息字典
    plot_countires:绘图库国家代码数据，字典格式，其中键为绘图库国家代码，值为对应的具体国名
    year:具体年份数据，以字符串格式程序，如"1970"
    map_file:输出的图片文件名
    目标：将指定某年的世界各国GDP数据在世界地图上显示，并将结果输出为具体的的图片文件
    提示：本函数可视化需要利用pygal.maps.world.World()方法
    """
    world_map_chart=pygal.maps.world.World()
    plot_countries, plot_countries_no_gdp, not_plot_countries=build_map_dict_by_name(gdpinfo, plot_countries,year)
    world_map_chart.title='全球GDP分布图'              #数据的可视化输出
    world_map_chart.add('%s'%year,plot_countries)
    world_map_chart.add('missing from world bank',plot_countries_no_gdp)
    world_map_chart.add('no data at this year',not_plot_countries)
    world_map_chart.render_to_file(map_file)

def test_render_world_map(year):  # 测试函数
    """
    对各功能函数进行测试
    """
    gdpinfo=dict(gdpfile="isp_gdp.csv", separator=",", quote='"', min_year=1960, max_year=2015,country_name="Country Name", country_code="Country Code")  # 定义数据字典
    pygal_countries=pygal.maps.world.COUNTRIES            #获得绘图库pygal国家代码字典
    render_world_map(gdpinfo, pygal_countries, year, "isp_gdp_world_name_1981.svg") # 测试时可以1970年为例，对函数继续测试，可以更改年份，将文件重新命名


# 程序测试和运行
print("欢迎使用世行GDP数据可视化查询")
print("----------------------")
year = input("请输入需查询的具体年份:")
test_render_world_map(year)
