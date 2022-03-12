---
title: Rails 下载文件
date: 2019-07-19 21:52:22
tags: Rails
categories: Ruby
---

虽然下载文件有 `Rails` 默认的 `send_data` 和 `send_file` 方法，还有像 [axlsx_rails](https://github.com/straydogstudio/axlsx_rails) 这样的第三方库。但是我仍然比较倾向于使用 `Spreedsheet XML` 的方式去开发下载文件的功能。

<!-- more -->

配置 `config/initializers/mime_types.rb` 文件，使 Rails 能够支持导出文件的后缀名。
```ruby
Mime::Type.register "application/csv", :csv
Mime::Type.register "application/xls", :xls
```

建立 `download` 方法，并配置好路由。
```ruby
# app/controllers/orders_controller.rb
def download
  @orders = Order.where(created_at: '2019-01-01'.to_time..'2019-02-01'.to_time)
  respond_to do |format|
    format.csv { send_data @orders.to_csv }
    format.xls { headers["Content-Disposition"] = 'attachment; filename=orders.xls' }
  end
end
```
```ruby
# app/models/order.rb
def self.to_csv(options = {})
  CSV.generate(options) do |csv|
    csv.tap do |csv_element|
      all.each do |order|
        csv_element << order.attributes.values_at(*column_names)
      end
    end
  end
end
```

创建 `app/views/orders/download.xls.erb` 文件，编写表格内容。
```ruby
<?xml version="1.0"?>
<Workbook xmlns="urn:schemas-microsoft-com:office:spreadsheet"
          xmlns:o="urn:schemas-microsoft-com:office:office"
          xmlns:x="urn:schemas-microsoft-com:office:excel"
          xmlns:ss="urn:schemas-microsoft-com:office:spreadsheet"
          xmlns:html="http://www.w3.org/TR/REC-html40">
  <Styles>
    <Style ss:ID="s62">
      <NumberFormat ss:Format="&quot;¥&quot;#,##0.00"/>
    </Style>
  </Styles>
  <Worksheet ss:Name="订单报表">
    <Table>
      <Row>
        <Cell><Data ss:Type="String">订单名称</Data></Cell>
        <Cell><Data ss:Type="String">订单数量</Data></Cell>
        <Cell><Data ss:Type="String">订单金额</Data></Cell>
        <Cell><Data ss:Type="String">购买用户</Data></Cell>
        <Cell><Data ss:Type="String">联系电话</Data></Cell>
        <Cell><Data ss:Type="String">订单创建时间</Data></Cell>
      </Row>
      <% @orders.each do |order| %>
        <Row>
          <Cell><Data ss:Type="String"><%= order.name %></Data></Cell>
          <Cell><Data ss:Type="Number"><%= order.count %></Data></Cell>
          <Cell ss:StyleID="s62"><Data ss:Type="Number"><%= order.amount %></Data></Cell>
          <Cell><Data ss:Type="String"><%= order.user.nickname %></Data></Cell>
          <Cell><Data ss:Type="String"><%= order.user.mobile %></Data></Cell>
          <Cell><Data ss:Type="String"><%= order.created_at.strftime('%Y-%m-%d %H:%M:%S') %></Data></Cell>
        </Row>
      <% end %>
    </Table>
  </Worksheet>
</Workbook>
```
现在启动你的 `Rails` 应用，访问 [http://localhost:3000/orders/download.xls](http://localhost:3000/orders/download.xls) 就可以导出订单报表。访问 [http://localhost:3000/orders/download.csv](http://localhost:3000/orders/download.csv) 可以导出 CSV 文件。

### 参考文档
- [Rails Guides](https://ruby-china.github.io/rails-guides/action_controller_overview.html#streaming-and-file-downloads)
- [http://railscasts.com/episodes/362-exporting-csv-and-excel](http://railscasts.com/episodes/362-exporting-csv-and-excel)
