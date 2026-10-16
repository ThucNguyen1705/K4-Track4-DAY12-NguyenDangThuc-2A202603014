# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Nguyễn Đăng Thực` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202603014` |
| GitHub | `ThucNguyen1705` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `split_seed 4236 / model_seed 394 (các seed 394, 395, 396, 397, 398)` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.247 (tất định) | 20.2 (583/2883, CI95 [18.8, 21.7]) | BLOCK (rớt R2, R3, R4, R6) |
| IDM | 1.517 (tất định) | 0.14 (4/2883, CI95 [0.05, 0.36]) | (baseline) |
| EgoMLP | 0.763–0.768 (TB 0.766 ± 0.002) | 12.8–35.8 (TB 20.7) | BLOCK (rớt R2, R3, R4, R5, R6) |
| PercMLP | 0.622–0.626 (TB 0.624 ± 0.001) | 0.5–2.6 (TB 1.3) | BLOCK (rớt R2, R3, R4, R5, R6) |
| PercMLP+ (sau khi sửa) | 0.817 ± 0.007 (TB qua 5 seed) | 0.24–0.31 (TB 0.29) | RELEASE (qua 7/7, nhưng R4 và R6 sát ngưỡng) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

`data_audit.json` cho thấy ba vấn đề. **(1) Cột `Acc_LV` không đáng tin:** tương quan giữa `Acc_LV` và đạo hàm `d(Speed_LV)/dt` chỉ **0.045**, trong khi cột tương ứng của xe sau (`Acc_FAV`) đạt **0.929**, nên `Acc_LV` gần như là nhiễu. Nếu dùng nó để khai phá ca phanh, em sẽ nhận sự kiện giả và bỏ sót ca thật. Em dùng `a_lead`: làm mượt `Speed_LV` (trung bình trượt 5 mẫu) rồi lấy sai phân trung tâm; phân vị p0.1 của gia tốc này là −2.77 m/s², hợp lý về vật lý. **(2) Trùng lặp:** trong **9993** quỹ đạo thô có **383** bản sao y hệt (cùng hồ sơ xe dẫn, xe sau, khoảng cách) nằm ở các ID cách xa nhau; loại đi còn **9610** quỹ đạo (992679 dòng). Nếu giữ lại, một bản rơi vào train và bản sao rơi vào test, tức test đã "thấy đáp án" và điểm sẽ đẹp giả. **(3) Rò rỉ khi chia:** log 10 Hz nên hai khung liền nhau của một quỹ đạo gần như giống hệt; chia theo khung sẽ đặt "hàng xóm" của mỗi khung test vào train. Vì vậy em chia **theo quỹ đạo** (seed 4236, 30 % test, được **2883** quỹ đạo giữ lại, rời hẳn tập train). Em cũng kiểm lại định nghĩa khoảng cách: `Spatial_Gap = Pos_LV − Pos_FAV − 4.5 m`, sai lệch tối đa 3.6e-11 m, nên cột gap dùng được.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã: 0/600 đoạn khác và 0 đoạn khác bit, nên công cụ phát lại tái lập được. Bản refactor float32 (đổi thứ tự phép tính) **khác bit ở 600/600 đoạn** nhưng **0/600 đoạn vượt dung sai** `atol = rtol = 1e-3`, sai lệch lớn nhất chỉ 4.0e-4 m/s². Lý do là phép cộng số thực không có tính kết hợp: `(0.1+0.2)+0.3 = 0.6000000000000001` nhưng `0.1+(0.2+0.3) = 0.6`, và cộng 50k số float32 theo hai chiều đã lệch tương đối 6.9e-6. So bằng `==` sẽ báo một lần dọn mã vô hại là hồi quy ở 100 % đoạn. Dung sai 1e-3 m/s² vẫn nhỏ hơn nhiều so với độ chính xác mà bộ chấp hành làm được.
Thay đổi AEB (lệnh −6 m/s² khi TTC < 1.5 s) làm lệch **11/600 đoạn** (82, 214, 306, 319, 360, …), sai lệch tối đa 2.0 m/s². Em mở các đoạn này bằng `peek.py` của GUIDE: chỉ có 34/60139 khung đổi lệnh, tất cả ở cuối đoạn khi xe đã tiến sát. Ví dụ đoạn 82, khung 75: gap 5.00 m, closing 3.39 m/s nên TTC 1.48 s, và lệnh IDM đổi từ −4.38 thành −6.0. AEB chỉ can thiệp khi TTC < 1.5 s *và* IDM đang phanh nhẹ hơn −6. Đây là thay đổi hành vi có chủ đích, khu trú ở các khung TTC thấp, nên phép so có dung sai bắt được nó mà không báo nhầm phần còn lại.
Phát "nhanh nhất" có 0 khung trễ và 0 khung rớt, vì thông điệp sau chỉ được gửi khi thông điệp trước đã xử lý xong, nên khái niệm hạn chót không tồn tại. Phát theo thời gian thật (thỉnh thoảng một khung mất 140 ms, vượt hạn 100 ms của một tick) cho thấy **586 khung trễ + 579 khung rớt trên 60139**, làm **381/600 đoạn** đổi lệnh vì lệnh cũ bị giữ lại. Như vậy phát nhanh nhất che mất lỗi trễ hạn tính toán, loại lỗi chỉ lộ ra trên xe thật.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới của em: tốc độ cruise **12.0 m/s** (suy từ MSSV), khoảng cách 2.76 → 59.31 m (p2..p98 của gap trong log), tốc độ tiến gần 0 → 12 m/s, tổng **60 × 45 = 2700 ô**. Ô **không bộ điều khiển nào tránh được** (phanh 9 m/s² sau độ trễ) chiếm **2.2 %** khi t_react = 0 và **7.4 %** khi t_react = 0.6 s, vì trong 0.6 s trễ xe mất thêm `dv × 0.6` m. Bộ mô phỏng khớp công thức: FullBrake đâm đúng 2.2 % / 7.4 % ô, với **0 ô lệch** khỏi biên ở cả hai độ trễ.
Khi tăng độ trễ, vùng đâm của IDM tăng từ **2.2 % lên 7.4 %**, trùng khít vùng bất khả kháng, nên phần va chạm "tránh được mà vẫn đâm" của IDM là **0.0 %** ở cả hai độ trễ: mọi va chạm của IDM thuộc loại "ai lái cũng đâm". Ngược lại, const-vel đâm 77.6 % ô, trong đó 70.2 % (t_react 0.6) là tránh được, tức là lỗi của chính bộ điều khiển. Biên bất khả kháng là thứ tách được hai trường hợp này.
Log **không phủ vùng khó**. Người lái trong log phanh mạnh nhất (p99.9) chỉ **3.6 m/s²**, trong khi bộ chấp hành làm được 9 m/s², và **8.6 %** ô của lưới đòi giảm tốc mạnh hơn mức đó. Em khai phá được 614 ca xe dẫn phanh (6.4 % quỹ đạo, phanh p50 −2.5, p99 −4.2 m/s²), nhưng trên `scenarios_idm.png` các chấm trắng dồn ở vùng tốc độ tiến gần thấp và không có chấm nào gần biên bất khả kháng ở góc "gap nhỏ, tiến gần nhanh". Mô hình học từ log vì thế chưa từng thấy nhãn phanh gấp.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng L2 | Va chạm vòng kín (TB qua seed) | Hạng va chạm |
|---|---|---|---|---|
| PercMLP | 0.624 | 1 | 1.3 % | 2 |
| EgoMLP | 0.766 | 2 | 20.7 % | 4 |
| const-vel | 1.247 | 3 | 20.2 % | 3 |
| IDM | 1.517 | 4 | 0.1 % | 1 |

Các cặp đổi thứ tự (`order_flips`): (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). Riêng cặp (const-vel, EgoMLP) đảo do chênh lệch rất nhỏ (20.2 % so với 20.7 % trung bình), nằm gọn trong dải seed 12.8–35.8 %, nên không vững. Các cặp có IDM thì đảo rõ ràng.
**IDM** có L2 tệ nhất (1.517 m) vì L2 đo mức *bắt chước một người lái cụ thể*. IDM có tham số cố định (T = 1.2 s, v0 = 15 m/s), còn người trong log giữ headway trung vị 1.86 s và lái theo phong cách riêng, nên sau 3 s vị trí lệch nhiều dù hành vi vẫn an toàn. Tuy nhiên IDM là luật **phản hồi**: lệnh phụ thuộc trực tiếp vào gap và tốc độ tiến gần (số hạng (s*/gap)² tăng vọt khi gap nhỏ), nên ở vòng kín mỗi sai số nhỏ được sửa ngay ở bước sau. Kết quả chỉ 4/2883 quỹ đạo va chạm, và 1.1 % trên lát xe dẫn phanh. Vòng hở không phân biệt được "khác phong cách" với "nguy hiểm".
**EgoMLP** chỉ thấy lịch sử tốc độ của chính nó. Trên log, lịch sử đó thuộc về người lái, mà người lái đã *phản ứng* với xe dẫn (bắt đầu giảm tốc). Ngoại suy quán tính vì thế cho L2 0.766 m, tốt hơn const-vel (1.247 m): mô hình ăn theo phản ứng của người. Ở vòng kín, lịch sử là của chính mô hình và nó không có gap hay tốc độ xe dẫn, nên không bao giờ nhận được tín hiệu để bắt đầu phanh. Ở seed xấu nhất, trung vị mức phanh lớn nhất chỉ 0.11 m/s², tỷ lệ va chạm 35.8 % và lên tới 62.8 % trên lát xe dẫn phanh. Đây là nhầm lẫn nhân quả cộng với sai số tích lũy.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Va chạm của EgoMLP qua 5 seed (394–398) lần lượt là **13.8, 35.8, 12.8, 20.4, 20.8 %**, tức dải **12.8–35.8 %**, chênh 23 điểm. Trong khi đó L2 vòng hở của cùng 5 seed chỉ dao động 0.763–0.768 m (std 0.002), nghĩa là vòng hở hoàn toàn không thấy sự bất ổn này. Khoảng Wilson 95 % của từng seed khá hẹp vì n = 2883: seed 396 là [11.7, 14.1] %, seed 395 là [34.1, 37.6] %, và **hai khoảng này không chồng nhau**. Wilson chỉ đo nhiễu lấy mẫu quỹ đạo với *một* mô hình đã cố định, không đo phương sai do huấn luyện (khởi tạo, thứ tự batch). Vì thế phải báo cả dải qua seed lẫn CI của seed xấu nhất, và đó cũng là cách cổng làm.
Nếu chỉ chạy seed 396, em sẽ báo "EgoMLP 12.8 % [11.7, 14.1], tốt hơn const-vel 20.2 %", trong khi seed 395 tệ hơn hẳn const-vel. Với PercMLP còn nguy hiểm hơn: seed 394 cho 0.49 %, cận trên 0.81 %, nhỏ hơn 0.14 % + 1 điểm, nên **riêng seed này qua được R2**, trong khi seed 396 là 2.6 % [2.1, 3.2]. Gặp một seed may mắn có thể dẫn tới phát hành một mô hình mà lần huấn luyện lại tiếp theo đâm nhiều hơn hẳn. Để so sánh, nhiễu cảm biến (gap ±1 m, tốc độ xe dẫn ±0.5 m/s) chỉ đẩy dải PercMLP lên 0.5–2.7 %, tức phương sai giữa các seed lớn hơn tác động của nhiễu.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Tỷ lệ ô lưới cắt đầu có va chạm bất kỳ, t_react = 0.6 s (trong ngoặc là va chạm phía sau):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng |
|---|---|---|
| const-vel | 77.6 % (sau 0.0 %) | 77.6 % (sau 0.0 %) |
| IDM | 75.7 % (sau 68.3 %) | 7.4 % (sau 0.0 %) |
| EgoMLP | 78.0 % (sau 0.0 %) | 78.0 % (sau 0.0 %) |
| PercMLP | 82.6 % (sau 57.4 %) | 25.1 % (sau 0.0 %) |
| PercMLP+ | 75.0 % (sau 67.2 %) | 9.3 % (sau 0.0 %) |

Với t_react = 0.0, IDM giảm từ 75.3 % xuống 2.2 % (va chạm sau từ 73.0 % xuống 0.0 %). Em hiệu chỉnh xe sau thành tài xế IDM với **T = 1.4 s**, chọn theo khoảng cách Wasserstein-1 nhỏ nhất giữa phân phối headway mô phỏng và của người (W1 = 0.34; T = 0.6 cho 0.52, T = 2.4 cho 0.87). Một xe không bao giờ phản ứng có W1 = 0.89, xa người lái thật hơn nhiều; headway trung vị của người là 1.86 s. Xe sau chạy cách 18.8 m và phản ứng trễ 0.6 s.
Xe sau "phát lại" giữ nguyên 12 m/s bất kể xe thử làm gì. IDM phanh *đúng* khi có xe chậm cắt vào, nên bị xe phát lại tông vào đuôi, nhưng ngoài đời người lái phía sau sẽ phanh theo. Đó là va chạm giả do thế giới không phản ứng. Trong thế giới phát lại, IDM (75.7 %) trông tệ ngang const-vel (77.6 %), còn const-vel không bao giờ bị tông đuôi chỉ vì nó không bao giờ phanh: thế giới phát lại vô tình *thưởng cho việc không phanh*, dễ dẫn tới kết luận "IDM không hơn gì không làm gì". Khi xe sau phản ứng, 7.4 % của IDM trùng khít vùng bất khả kháng 7.4 %, tức IDM không có lỗi, và thứ tự đúng IDM < PercMLP+ < PercMLP < const-vel/EgoMLP hiện ra. Vì vậy cổng (R6) dùng thế giới phản ứng.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi 5 quy tắc: R2 (seed xấu nhất 2.6 %, cận trên 3.2 % > 0.1 % + 1 điểm), R3 (lát phanh 8.0 %, cận trên 12.7 % > 1.1 % + 5 điểm), R4 (17.7 % lưới cắt đầu tránh được mà vẫn đâm, so với 0.0 % + 2 điểm), R5 (dải 0.5..2.6 %, rộng hơn 1 điểm) và R6 (25.1 % so với 7.4 % + 2 điểm). Có hai nguyên nhân gốc. **(a) Thiếu kịch bản** dẫn tới rớt R4 và R6: log không có cắt đầu ở cự ly gần với tốc độ tiến gần lớn (8.6 % lưới đòi phanh mạnh hơn 3.6 m/s², mức mạnh nhất người trong log từng phanh), nên mô hình chưa từng thấy nhãn phanh gấp. Ở tốc độ 20 m/s chưa từng huấn luyện, PercMLP đâm 25.4–29.9 % lưới. **(b) Sai số tích lũy (covariate shift)** dẫn tới rớt R2, R3 và R5: L2 vòng hở chỉ 0.624 m, nhưng ở vòng kín mô hình tự đi vào những trạng thái mà người lái không bao giờ đi vào, nên không có nhãn để học cách sửa, và mức độ trôi phụ thuộc vào seed.
Em sửa bằng hai cách. **(1)** Thêm cửa sổ huấn luyện từ 20000 kịch bản cắt đầu mô phỏng, gán nhãn bằng IDM, để lấp đúng vùng lưới mà log thiếu (chữa a). **(2)** Chạy 2 vòng DAgger (`dagger_windows`): mô hình tự lái trên log huấn luyện, ghi lại các trạng thái *nó tự đi vào*, rồi gán nhãn bằng dự báo 3 s của IDM ngay tại các trạng thái đó. Cách này dạy mô hình hồi phục từ lỗi của chính nó (chữa b). Sau khi sửa, PercMLP+ đạt R2 0.3 % (cận trên 0.6 %), R3 1.1 % (cận trên 3.8 %), R4 1.8 %, R5 0.2..0.3 %, R6 9.3 %, và được **RELEASE** với 7/7 quy tắc.
PercMLP+ **không tốt hơn IDM**, chỉ *không kém hơn trong biên cho phép*: va chạm trên log 0.24–0.31 % so với IDM 0.14 %, lưới cắt đầu 1.8 % so với 0.0 %, thế giới phản ứng 9.3 % so với 7.4 %, lát phanh bằng nhau ở 1.1 %. Nó học nhãn từ IDM nên tối đa chỉ xấp xỉ được thầy. Hai quy tắc qua sát nút: R4 là 1.8 % so với ngưỡng 2 %, R6 là 9.3 % so với ngưỡng 7.4 + 2 %. L2 vòng hở còn *xấu đi*, từ 0.624 lên 0.817 m, vì nhãn giờ là IDM chứ không phải người; đây thêm một bằng chứng rằng L2 không đo an toàn. Em đề xuất **chưa thay IDM**. Trước khi quyết định, đội cần: (i) nêu rõ lợi ích PercMLP+ mang lại so với IDM, vì cổng chỉ chứng minh "không kém"; (ii) thêm seed và kịch bản (tốc độ cao hơn, xe dẫn phản ứng, độ trễ cảm biến), vì R4 và R6 quá sát ngưỡng và ở 20 m/s mô hình vẫn đâm 0.1–1.7 % lưới, trong khi IDM 0.0 %; (iii) chạy shadow mode trên xe (mô hình chỉ tính lệnh, IDM vẫn lái) và đối chiếu; (iv) giữ IDM làm lớp an toàn dự phòng.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

**(1) Thế giới mô phỏng là giả định của chính chúng ta.** Mô phỏng chỉ có một chiều dọc, xe coi như điểm, xe dẫn trong log *không phản ứng* với xe thử, và xe sau là IDM với T = 1.4 s được hiệu chỉnh trên đúng một thống kê (W1 headway 0.34, chưa bằng 0). Các con số như 0.24–0.31 % hay 9.3 % chỉ đúng trong thế giới này. Đường thật có chuyển làn, người đi bộ, mặt đường ướt (b_max không phải 9 m/s²). Mọi sai số trong cách mô hình hóa đều chuyển thẳng thành rủi ro của quyết định phát hành.
**(2) Không có cảm biến thật.** Gap và tốc độ xe dẫn được đưa thẳng vào mô hình; bài lab chỉ thử nhiễu Gauss (±1 m, ±0.5 m/s) và độ trễ cố định 0.6 s. Các lỗi của perception thật như nhận dạng sai, mất bám, phát hiện trễ đều chưa được kiểm tra, mà đây lại là chỗ mô hình học yếu nhất. Ngay ở bước 1, chỉ riêng gai tính toán đã gây 586 khung trễ và 579 khung rớt.
**(3) Phân phối và cỡ mẫu.** Dữ liệu là Argoverse 2 trong đô thị, mỗi quỹ đạo 70–109 bước, tốc độ xe dẫn p99 là 15.6 m/s. Với 2883 quỹ đạo test, IDM có 4 va chạm và CI [0.05, 0.36] %, nên không thể khẳng định tỷ lệ sự cố hiếm như yêu cầu của xe thật. Ngoài phân phối (20 m/s), PercMLP+ đã đâm 0.1–1.7 % lưới.
**(4) Cổng mang tính vòng tròn.** IDM vừa là "thầy" gán nhãn vừa là baseline của cổng, nên "qua cổng" chỉ có nghĩa là bắt chước thầy đủ tốt. Cổng không nói gì về những tình huống mà chính IDM cũng sai. Các biên 1, 2, 5 điểm là lựa chọn của đội chứ không suy ra từ yêu cầu an toàn.
**Việc cần làm tiếp:** dùng tác nhân phản ứng cho cả xe dẫn và các xe xung quanh (mô phỏng 2D, có chuyển làn); mô phỏng cảm biến với lỗi thực tế; mở rộng lưới kịch bản theo tốc độ và điều kiện mặt đường; tăng số seed và cỡ mẫu (tính trước số km cần để đạt cận trên mong muốn). Sau đó chạy shadow mode và thử trên đường kín trước khi cho lái thật.
