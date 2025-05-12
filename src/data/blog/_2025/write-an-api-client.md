---
title: Viết một API client trong Ruby on Rails
author: nukun
pubDatetime: 2025-05-12T22:15:00Z
slug: write-an-api-client
featured: true
draft: false
tags:
  - work
  - api
  - client
  - ruby
  - rails
<!-- ogImage: ../../assets/images/forrest-gump-quote.png -->
description: Vừa làm vừa học. Ôn lại cách viết API client trong Ruby on Rails.
<!-- canonicalURL: https://example.org/my-article-was-already-posted-here -->
---

## Vấn đề
Hệ thống tôi đang làm tạm gọi là *DUCK* đi, gọi API bên thứ 3 để lấy lịch trình tàu và dựa vào đó để lên danh sách các ứng cử viên cho kế hoạch kiểm tra. Từ lịch trình sẽ hiển thị 3 thông tin là **port** (cảng), **ETB** (thời gian cập cảng dự kiến) và **ETD** (thời gian rời cảng dự kiến) kèm với vessel code (mã tàu) để liên kết với các tàu đăng ký trong hệ thống. Ngoài các thông tin trên còn hiển thị thêm thời gian kiểm tra tàu gần nhất, dự kiến ngày kiểm tra tiếp theo, VRS (điểm rủi ro của tàu) và yêu cầu từ bên khách hàng. Bảng thông tin này có phân trang, tất cả các cột ngoại trừ `remarks` và yêu cầu của khách hàng đều có thể sắp thứ tự. Có chức năng tìm kiếm theo tên và IMO của tàu, kèm theo checkbox có bật tính năng lọc các con tàu có thời gian kiểm tra dự kiến sau tháng hiện tại. (Giống như là tôi đang kể cả dự án...)

Logic hiện tại là khi mở trang, ô chọn tháng có giá trị mặc định là tháng hiện tại. Gọi Ajax lên server lấy danh sách vessel ID có lịch trình trong tháng với các cảng áp dụng cho DUCK. Sau đó khởi tạo Datatables với data gọi từ ajax kèm theo các tham số tương ứng như không hiển thị tàu đã ẩn, yêu cầu bên khách hàng, cảng, tháng, và vessel ID đã lấy trước đó. Server query DB để lấy danh sách tàu theo vessel ID và các filter từ Datatable sau đó gọi API lấy lịch trình tàu rồi gắn vào từng con tàu trong hệ thống để có được 3 thông tin đã kể ở trên. Tiếp đến sẽ có một service tính toán VRS riêng dựa theo con tàu. Rồi lọc theo kì kiểm tra dự kiến. Xong xuôi các bước xử lí dữ liệu thì tới sắp thứ tự theo cột mà khách hàng muốn, rồi phân trang, định dạng theo mong muốn của Datatable rồi trả về hiển thị.

Như vậy có tới 2 lần gọi API trong 1 request này. Chưa kể mỗi lần thao tác sắp xếp, chuyển trang, tìm kiếm đều phải gọi API và xử lý dữ liệu trên bộ nhớ. Khá là chậm và tốn tài nguyên. Ưu điểm là luôn lấy dữ liệu mới nhất từ API về.

## Nên chọn cách nào?
Ban đầu tôi dự định cache lịch trình từ API và lưu vào database để thực hiện tính toán, phân trang, thứ tự bằng database cho lẹ nhưng lại vướng tới tính VRS. Tính này khá là phức tạp vì nó tổng hợp nhiều điểm, mỗi điểm lại tham chiếu tới bảng khác để lấy ra khoảng điểm tương ứng với điểm. Nếu thông tin của vessel nằm trong khoảng nào thì lấy điểm đó và cuối cùng cộng lại để ra điểm VRS. Nhưng chưa xong, còn phải tính ngày dự kiến kiểm tra, ngày này dựa vào điểm rủi ro, và tiếp tục điểm này phụ thuộc vào VRS.

Nói chung sẽ rất tốn thời gian để thực hiện kế hoạch này cũng như việc dời logic tính toán điểm về phía DB. Nó làm phân tán logic, DB chỉ nên thực hiện các logic lấy kết quả phù hợp và hiệu quả còn logic này thuộc phàm trù hệ thống nên nằm trong code Ruby để dễ quản lý.

Vẫn còn phương án là cache lịch trình vào Redis thay vì DB, như vậy sẽ đỡ phải gọi API thường xuyên tốn nhiều thời gian. Chỉ ngại một điều là nên để thời gian hết hạn là bao nhiêu? Vì khi lên lịch thì khách hàng cần thông tin mới nhất từ IBIS để ra quyết định hợp lý. Nếu cache thì dữ liệu sẽ bị trễ một nhịp với IBIS. Bảng ứng viên này thường tốn 1~3 giây để tải. Có nên đánh đổi nó với việc tốn dưới 1 giây nhưng dữ liệu trễ một chút không?

Những người quản lý dự án không quan tâm tới việc nhanh hay chậm lắm, mà đúng ra thì họ cũng không quan tâm nhiều tới dự án vì họ lo những dự án lớn hơn còn hệ thống này thì nhỏ, khách hàng ít. Nếu tôi triển khai cache thì phải qua mấy vòng hỏi và trả lời còn không biết có được chấp thuận không. Đầu tiên là hỏi lead VN, giải thích tại sao phải làm cách này, ảnh hưởng những đâu, test nhiều không, làm lâu không. Sau đó lại đi hỏi quản lý dự án và giải thích bằng tiếng anh. Anh này thì không cứng chuyên môn cũng như dự án vì ảnh mới đảm nhận. Nên việc giải thích rất là mất thời gian và công sức. Và còn đội BA, QC nữa. Rắc rối chút ít tôi còn làm được chứ rắc rối nhiều như này tôi thấy "no hope" quá. Nên thôi, chỉ đổi API thôi còn logic vẫn giữ như cũ nhé.

## Triển khai như thế nào?
Cần một class có thể gửi lời nhắn `fetch_vessel_schedules` tới object của nó kèm tham số cần thiết để yêu cầu nó gọi API và trả về lịch trình. Ví dụ như:
```ruby
client = Api::VesselSchedule::Client.new
response = client.fetch_vessel_schedules(params)
response.schedules
```

Các yếu tố cần xem xét là params truyền vào, thiết kế class cha, xử lý lỗi, định dạng, lọc dữ liệu trả về.
### Class cha
Ngoài API cho lịch trình tàu hệ thống còn gọi thêm vài API khác cho mục đích khác nhau. Có thể trong tương lai sẽ đổi hết sang endpoint mới này nên cần thiết phải thiết kế một class cha với các yếu tố chung cơ bản làm sườn. Các thông tin chung liên quan tới endpoint mới là URL gốc (`https://example.com/api/` chẳng hạn), API token. 2 thông tin này được lưu trong biến môi trường và khai báo bằng hằng số trong class cha. Thông tin chung khi khởi tạo thì sao? Liên quan tới request thì chắc là `timeout` (thời gian chờ), cho phép khi khởi tạo client có thể cung cấp timeout tuỳ ý nhưng sẽ có giá trị mặc định để không phải lúc nào cũng phải khai báo kèm.

```ruby
class Api::BaseClient
  BASE_URL = ENV['BASE_API_URL']
  API_TOKEN = ENV['API_TOKEN']

  attr_reader :timeout

  def initialize(timeout = 10)
    @timeout = timeout
  end
end
```

Rồi đi sau vào cách thực hiện. Quay trở lại cách gọi ban đầu `fetch_vessel_schedules` cần làm những gì? Đầu tiên là tạo một request. Hiện tại hệ thống dùng `Typhoues` để thực hiện HTTP request nên request sẽ là một object của Typhoues. Request cần có path (đường dẫn cụ thể), phương thức (GET, POST,...) và params (nếu là GET) hoặc body (nếu là POST). Và đừng quên kèm các header cần thiết.

Vì đa phần sẽ gọi API để nhận dữ liệu chứ không thay đổi resource nguồn nên để mặc định phương thức là GET. Path sẽ định nghĩa trong từng class con vì nó cố định theo class, có thể bỏ nó trong hằng số hoặc method. Params đầu vào cần phải được định dạng và kiểm tra, nếu nó valid (hợp lệ) thì mới cho phép thực hiện request còn không thì phải báo lỗi.

```ruby
def build_request(path, method: :get, params: {}, body: nil)
  options = { method: method,
              headers: default_headers,
              timeout: timeout }
  # Tuỳ vào mỗi loại client mà có cách định dạng params khác nhau
  if method == :get && params.present?
    options[:params] = params
  end
  # Nếu post,... thì gửi option body cho Typhoes
  if [:post, :put, :patch].include?(method) && body.present?
    options[:body] = body.is_a?(String) ? body : body.to_json
  end

  Typhoeus::Request.new("#{BASE_URL}#{path}", options)
end
```

Tới bước này bắt đầu phát sinh những chuyện ngoài ý muốn mà cha cần phải xử lý hộ thay mấy thằng con.
### Báo lỗi (cho tụi con biết mà xử lý)
Đã tạo request rồi thì phải thực thi nó `execute_request`. Có 2 việc cần làm khi thực thi request đó là thực sự gọi nó (`run`) và kiểm tra lỗi (`check_response`). Khi một hành động xảy ra liên quan tới bên ngoài thì sẽ phát sinh nhiều vấn đề mà ta có thể biết và không biết tuỳ thuộc ta hiểu đối phương nhiều hay ít nhưng chắc chắc là nếu có vấn đề thì bên ta phải có cách nào để để thông báo cho hệ thống hoặc chuyển hướng lỗi tới nơi khác theo các kiểu định sẵn mà hệ thống hiểu rõ. Cụ thể hơn nếu response thành công thì trả về response, nếu thất bại thì la làng lên để nơi nào đó thấy và xử lý lỗi. Các lỗi có thể gặp là quá timeout, không thể nhận được response vì lí do gì đó, hoặc một lỗi cụ thể từ phía server bên kia gửi về.

Để hệ thống có thể bắt được các lỗi thuộc về API này thì trước hết cần tạo và đặt một cái tên thống nhất. Khi đó ở bất kì nơi nào gọi API đều có thể bắt chính xác lỗi này để xử lý. Ta có thể đặt tên là `ApiError`. Nên nhớ các lỗi tuỳ biến này đều kế thừa từ `StandardError`.

```ruby
def execute_request(request)
  response = request.run
  check_response(response)
end

def check_response(response)
  if response.success?
	response
  elsif response.timed_out?
	raise ApiError.new("Request timed out", response)
  elsif response.code.zero?
	raise ApiError.new("Could not get an HTTP response (#{response.return_message})", response)
  else
	error_message = "HTTP request failed: #{response.code}"
	raise ApiError.new(error_message, response)
  end
end
```

Đến đây các lớp con có thể có thể gọi request để lấy về response

```ruby
request = build_request(PATH, params: params)
response = execute_request(request)
```

### Xử lý dữ liệu trả về
#### Hai tình huống
Tưởng tượng IBIS cung cấp những khúc gỗ nhỏ và ta cần chúng cho mục đích riêng của mình. Khi nhận được những khúc gỗ này ta cần phải làm 2 việc đó là mài dũa nó sao cho tương thích, có nghĩa là hệ thống có thể nhận diện và dùng được (hình vuông, mài nhẵn, hoặc hình trụ,...) và loại bỏ những khúc gỗ ta không cần (có thể do quá dài, bị hư hại, đã cũ,...).

Câu hỏi nảy sinh là nên làm bước nào trước? Loại bỏ hay mài dũa trước? Hãy tiếp tục tưởng tượng có một cái máy dùng riêng cho việc loại bỏ. Khúc gỗ sẽ đi qua máy và máy phải nhận diện được khúc gỗ rồi mới kiểm tra chất lượng để quyết định xem nên loại hay giữ. Nếu máy dùng chung bộ nhận diện với hệ thống thì sẽ không thể loại được gỗ vì những khúc gỗ lúc này còn méo mó, lồi lõm theo hình dạng nơi nó sinh ra. Vậy thì ta phải dùng máy có bộ nhận dạng giống với bên IBIS. Nhưng bộ nhận diện này đã nằm trong một máy mài rồi. Nó sẽ biết hình dạng ban đầu của khúc gỗ và mài ra hình dạng mà hệ thống mong muốn (có thể nhận dạng được). Ở đây xảy ra việc phân tán, và ôm đồm quá nhiều công việc. Có thể nghĩ tới một cách giải quyết là định nghĩa một module nhận dạng từ bên IBIS sang bên hệ thống mình. Và đưa nó vào 2 máy cho nó đọc.

Giờ nghĩ tới cách còn lại là mài trước và lọc sau. Khi đó máy mài sẽ nắm kiến thức nhận dạng từ bên IBIS để biến nó về phù hợp với hệ thống sau đó máy lọc sẽ lọc ra dựa trên những khúc gỗ đã được mài mà hệ thống nhận dạng được. Sẽ đơn giản hơn với cách ban đầu. Nhưng đổi lại khi mài sẽ mất nhiều thời gian hơn vì nó mài cả những khúc gỗ chưa được lọc. Tới bước này rõ ràng cần chạy một bài kiểm tra để xem bên nào nhanh hơn.

#### Bài kiểm tra
⚠️ *Chờ viết code so sánh...*
Thế nên tôi chọn mài trước lọc sau. Theo lẽ thường của OOP thì nên có 2 class một cho việc mài, một cho việc lọc. Nhưng tôi sẽ chỉ tách ra class mài vì sau này sẽ gom nhiều logic liên quan vào nó. Mài trong hệ thống sẽ là `Parser`. Còn lọc thì sẽ nằm chung với class `Response`, nó chịu trách nhiệm xử lý response.

```ruby
class Parser
  def self.parse_schedules(data)
    return [] if data.blank?

    json_data = data.is_a?(String) ? JSON.parse(data) : data
    schedules_data = json_data["list"] || []

    schedules_data.map do |schedule|
      {
        imo: schedule["imoNumber"],
        vessel_code: schedule["vesselCode"],
        etb: parse_datetime(schedule["etbDate"]),
        # ...
      }
    end
  end

  def self.parse_datetime(datetime_string)
    return nil if datetime_string.blank?

    DateTime.strptime(datetime_string, "%Y%m%d%H%M")
  end
end
```

```ruby
class Response
  attr_reader :raw_response, :params, :schedules

  def initialize(response, params = {})
    @raw_response = response
    @params = params
    @schedules = filter_schedules
  end

  def success?
    raw_response.success?
  end

  def total_count
    schedules.length
  end

  private

  # Đây là mài
  def parsed_schedules
    @parsed_schedules ||= VesselSchedule::Parser.parse_schedules(raw_response.body)
  end

  ## Lọc ở đây, có thể tách ra thành class riêng nếu phức tạp
  def filter_schedules
    parsed_schedules.select { |schedule| valid_schedule?(schedule) }
  end
end
```

Đến đây thì chúng ta có thể dùng được rồi.

## Nhớ bắt lỗi
Dùng như ban đầu. Nhưng nên nhớ phải xử lý lỗi.
```ruby
begin
  client = Api::VesselSchedule::Client.new
  response = client.fetch_vessel_schedules(params)
  response.schedules
rescue ApiError => e
  # Xử lý lỗi ở đây, có thể log hoặc gửi thông báo lên kênh thông báo
  []
end
```

## Suy nghĩ, cảm nhận, tâm tư tình cảm,...
Bài viết đầu tiên nhưng cũng khá dài. Dạo này làm mà cứ lơ ngơ lớ ngớ nên thôi ráng viết lại cho đầu óc thông thoáng suy nghĩ trôi chảy hơn. Để xem xem mấy anh QC test ra lỗi nhiều không (chớ sao mà không lỗi được!). Các bạn thấy chỗ nào chưa thoả đáng hay cần phát triển thêm thì bảo mình nhé vì mình cũng vừa làm vừa học thôi.

Cảm ơn đã đọc bài! 🍻
