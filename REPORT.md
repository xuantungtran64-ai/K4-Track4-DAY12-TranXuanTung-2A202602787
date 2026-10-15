# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Trần Xuân Tùng` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602787` |
| GitHub | `xuantungtran64-ai` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `5749 / 7376` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.248 | 20.2% | BLOCK |
| IDM | 1.518 | 0.0% | (baseline) |
| EgoMLP | 0.762 ± 0.003 | 16.5..24.1% | BLOCK |
| PercMLP | 0.621 ± 0.003 | 0.7..3.0% | BLOCK |
| PercMLP+ (sau khi sửa) | 0.813 ± 0.024 | 0.2..0.3% | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong Bước 0, quá trình kiểm toán dữ liệu tại evidence/data_audit.json đã phát hiện hai vấn đề nghiêm trọng:
1. Có 383 quỹ đạo trùng lặp tuyệt đối (exact duplicates) bị loại bỏ, đưa tổng số quỹ đạo từ 9993 quỹ đạo thô xuống còn 9610 quỹ đạo sạch (với 992679 dòng). Các quỹ đạo trùng lặp này xuất hiện ở các Trajectory_ID cách xa nhau, nếu không khử trùng lặp thì khi chia tập ngẫu nhiên sẽ gây rò rỉ dữ liệu (data leakage) giữa tập train và test.
2. Cột gia tốc thô Acc_LV của xe dẫn có hệ số tương quan với đạo hàm vận tốc (dv/dt) chỉ đạt 0.045, gần như bằng 0 (trong khi cột Acc_FAV của xe theo sau đạt tương quan tới 0.929). Điều này cho thấy cảm biến gia tốc xe dẫn bị nhiễu cực mạnh và hoàn toàn không đáng tin cậy. Do đó, hệ thống bắt buộc phải thay thế bằng đại lượng a_lead được tính bằng sai phân hữu hạn trung tâm từ chuỗi vận tốc đã làm mượt (v_lead_s) với cửa sổ lọc trung bình trượt 5 bước (width=5, dt=0.1s).
3. Việc phân chia tập dữ liệu bắt buộc phải thực hiện theo đơn vị từng quỹ đạo (trajectory) với tỷ lệ train/test là 70/30 (6727 train / 2883 test theo seed 5749) thay vì chia theo từng khung hình (frame), bởi vì các khung hình liên tiếp trong cùng một quỹ đạo 10 Hz có mức độ tự tương quan cực kỳ cao; chia theo khung hình sẽ dẫn đến việc mô hình chỉ học vẹt các bước lân cận và đánh giá quá lạc quan năng lực thực tế.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Từ kết quả chạy lại nhật ký ở Bước 1 trong evidence/replay.json trên 600 đoạn log (9.5 MB):
1. Bản refactor tối ưu sang float32 và thay đổi thứ tự cộng khác bit ở toàn bộ 600/600 đoạn (100%), tuy nhiên số đoạn vượt dung sai (atol=1e-3, rtol=1e-3) là đúng 0/600 đoạn, với sai lệch tuyệt đối lớn nhất max_abs_diff chỉ đạt 1.22e-04 m/s^2. Điều này khẳng định phép so sánh có dung sai là hoàn toàn cần thiết để tránh báo lỗi giả khi bản chất thuật toán không đổi nhưng chịu ảnh hưởng của tính chất không kết hợp số thực: (0.1+0.2)+0.3 = 0.6000000000000001 khác với 0.1+(0.2+0.3) = 0.6.
2. Phiên bản thay đổi hành vi (thêm phanh khẩn cấp AEB với ngưỡng TTC < 1.5s) chỉ làm lệch đúng 6/600 đoạn (cụ thể là các đoạn chỉ số [191, 214, 252, 358, 450, 521]). Khi mở các đoạn này ra quan sát, sự sai lệch chỉ xuất hiện cục bộ tại các bước thời gian mà xe dẫn phanh đột ngột khiến khoảng cách co hẹp nhanh và TTC tụt xuống dưới 1.5s, kích hoạt lệnh phanh tối đa a = -9.0 m/s^2 thay vì mức phanh nhẹ của IDM thông thường.
3. Chế độ phát lại thời gian thật (realtime) làm lộ ra 583 khung bị trễ (late) và 577 khung bị rớt (dropped) trên tổng số 59838 khung hình (khiến 383/600 đoạn bị thay đổi kết quả), điều mà chế độ phát nhanh nhất (fastest) che giấu hoàn toàn (0 late, 0 dropped). Trong hệ thống tự hành thực tế, các độ trễ xử lý và mất gói tin cảm biến sẽ ảnh hưởng trực tiếp đến lệnh điều khiển phanh.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Dựa trên bằng chứng tại evidence/scenarios.json:
1. Lưới kịch bản của tôi được cấu hình ở vận tốc cruise v_ego = 14.0 m/s, gồm 60 mức khoảng cách (từ 2.37 m đến 66.86 m) và 45 mức tốc độ tiến gần (từ 0.0 m/s đến 14.0 m/s), tạo thành không gian gồm 2700 ô kịch bản.
2. Về biên giới hạn vật lý bất khả kháng (unavoidable): ở độ trễ phản ứng t_react = 0.0s, có 3.6% số ô (98 ô) là không thể tránh khỏi va chạm dù phanh tối đa b_max = 9.0 m/s^2. Khi độ trễ tăng lên t_react = 0.6s, vùng bất khả thi tăng vọt lên 10.2% số ô (276 ô), tức tăng thêm 6.6 điểm phần trăm (tăng gần gấp 3 lần diện tích va chạm vật lý). Kết quả kiểm chứng bộ mô phỏng khớp hoàn toàn với biên vật lý lý thuyết với đúng 0 ô sai lệch ở cả hai độ trễ.
3. Vùng va chạm của baseline IDM tăng từ 3.6% (ở 0.0s) lên 10.2% (ở 0.6s), trùng khít 100% với biên bất khả thi (tức diện tích va chạm do lỗi bộ điều khiển attributable_area bằng đúng 0.0%).
4. Về độ bao phủ của log tự nhiên: Người lái xe trong log chỉ phanh mạnh nhất đạt 3.62 m/s^2 (tính tại phân vị 99.9 của gia tốc người), trong khi bộ chấp hành có thể đạt 9.0 m/s^2. Có tới 12.7% diện tích lưới (342 ô) đòi hỏi mức giảm tốc vượt quá khả năng phanh mạnh nhất của bất kỳ tài xế nào trong log. Toàn bộ 614 sự kiện phanh khai thác từ log (chiếm 6.4% tổng số quỹ đạo) chỉ có mức giảm tốc trung vị p50 là -2.53 m/s^2 và p99 là -4.21 m/s^2, hoàn toàn không bao phủ được các tình huống xe cắt đầu ở cự ly hẹp.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh 4 bộ điều khiển từ evidence/closedloop.json:

| Bộ điều khiển | L2 vòng hở [m] | Hạng L2 | Va chạm vòng kín [%] | Hạng vòng kín |
|---|---|---|---|---|
| const-vel | 1.248 | 3 | 20.2% | 3 |
| IDM | 1.518 | 4 | 0.0% | 1 |
| EgoMLP | 0.762 | 2 | 20.4% | 4 |
| PercMLP | 0.621 | 1 | 1.6% | 2 |

1. Các cặp đảo ngược thứ hạng (ranking inversion): Có 4 cặp bị đảo ngược hoàn toàn gồm: ('const-vel', 'IDM'), ('const-vel', 'EgoMLP'), ('IDM', 'EgoMLP'), ('IDM', 'PercMLP').
2. Nghịch lý IDM: IDM có điểm L2 vòng hở kém nhất (1.518 m, đội sổ hạng 4) nhưng vào mô phỏng vòng kín lại lái an toàn nhất (0.0% va chạm trên log, hạng 1). Nguyên nhân là hàm mục tiêu của IDM không phải là tối thiểu hóa sai số bắt chước vi mô từng mét di chuyển của tài xế con người, mà nó liên tục điều chỉnh gia tốc dựa trên khoảng cách tức thời và khoảng cách an toàn mong muốn. Khi vào vòng kín, khả năng phản hồi theo trạng thái thực tế giúp IDM triệt tiêu hoàn toàn sai số tích lũy.
3. Nghịch lý EgoMLP: EgoMLP có L2 vòng hở rất ấn tượng (0.762 m, tốt hơn nhiều so với 1.248 m của const-vel), nhưng trong vòng kín lại đâm nhiều nhất (20.4%, seed xấu nhất lên tới 24.1%, trên lát cắt phanh đâm 53.2%). Nguyên nhân là EgoMLP chỉ được cung cấp thông tin vận tốc bản thân trong quá khứ mà hoàn toàn không có cảm biến nhận thức về khoảng cách (gap) hay vận tốc xe dẫn (lead_v). Khi xe trước phanh gấp, EgoMLP hoàn toàn mù về chướng ngại vật phía trước và tiếp tục duy trì tốc độ dẫn đến va chạm.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Phân tích tính ổn định qua 5 hạt giống ngẫu nhiên (seeds [7376, 7377, 7378, 7379, 7380]) từ evidence/closedloop.json:
1. Độ dao động của EgoMLP: Tỷ lệ va chạm vòng kín của EgoMLP trải dài từ 16.5% đến 24.1% trên 2883 quỹ đạo kiểm thử (dải dao động rộng tới 7.6 điểm phần trăm qua 5 seed: [24.1%, 16.5%, 18.7%, 20.0%, 22.8%]). Đối với PercMLP, tỷ lệ va chạm cũng dao động từ 0.7% đến 3.0% (chênh lệch 2.3 điểm phần trăm).
2. Khoảng tin cậy Wilson 95%: Tại hạt giống xấu nhất của EgoMLP (24.1%), khoảng tin cậy Wilson 95% là [22.6%, 25.7%]. Trong khi đó ở hạt giống tốt nhất (16.5%), khoảng tin cậy là [15.2%, 17.9%]. Hai khoảng tin cậy này hoàn toàn tách rời nhau và không hề chồng lấn!
3. Nguy cơ khi chỉ chạy 1 seed: Nếu một nhóm kỹ sư chỉ huấn luyện và đánh giá trên một hạt giống ngẫu nhiên duy nhất, họ có thể tình cờ chọn trúng seed may mắn (16.5%) và vội vã kết luận mô hình đạt độ an toàn chấp nhận được, hoặc chọn trúng seed xấu (24.1%) và loại bỏ một kiến trúc tiềm năng. Nguy hiểm hơn, việc chỉ báo cáo một con số duy nhất sẽ che giấu hiện tượng bất ổn định trong tối ưu hóa phi lồi của mạng nơ-ron sâu, khiến hệ thống tự hành khi đưa ra thực tế có thể rơi vào vùng trọng số hoạt động cực kỳ nguy hiểm mà không hề được cảnh báo trước.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng so sánh từ evidence/agents.json (ở độ trễ t_react = 0.6s trên lưới 2700 ô kịch bản):

| Bộ điều khiển | Xe sau phát lại (Any / Rear) | Xe sau phản ứng (Any / Rear) |
|---|---|---|
| const-vel | 80.6% / 0.0% | 80.6% / 0.0% |
| IDM | 80.9% / 70.6% | 10.2% / 0.0% |
| EgoMLP | 83.9% / 0.0% | 83.9% / 0.0% |
| PercMLP | 77.6% / 44.1% | 33.8% / 0.0% |
| PercMLP+ | 80.4% / 69.7% | 11.3% / 0.0% |

1. Hiệu chỉnh tham số T: Qua quét khoảng cách Wasserstein-1 (W1) trên 3000 quỹ đạo, giá trị tối ưu được chọn là T = 1.4s với khoảng cách W1 nhỏ nhất đạt 0.36 (so với khoảng cách thời gian trung vị của tài xế con người trong log là 1.86s). Một tác nhân không phản ứng (const-vel) có khoảng cách W1 lên tới 0.86, chứng minh xe sau mô hình hóa bằng IDM phản ánh chân thực hơn hẳn phân phối khoảng cách bám đuôi thực tế.
2. Va chạm giả (Artifact of Non-reactive World): Khi xe sau chỉ phát lại cứng nhắc tốc độ trong log, nếu xe thử nghiệm IDM phanh gấp để tránh xe cắt đầu phía trước, xe sau không nhận biết được việc đó và đâm thẳng vào đuôi xe thử nghiệm, tạo ra tới 70.6% số ô va chạm từ phía sau (rear collisions)! Ngược lại, bộ điều khiển tồi như const-vel không bao giờ phanh nên xe sau không bao giờ đâm nó (rear = 0.0%).
3. Thay đổi kết luận: Trong thế giới phát lại, IDM trông tệ ngang const-vel (80.9% vs 80.6% va chạm). Nhưng khi chuyển sang thế giới phản ứng, va chạm đuôi của IDM giảm từ 70.6% về đúng 0.0%, đưa tỷ lệ va chạm thực sự của IDM xuống chỉ còn 10.2% (bằng đúng biên bất khả thi). Điều này khẳng định môi trường mô phỏng bắt buộc phải có các tác nhân biết phản ứng mới đánh giá đúng bản chất an toàn của thuật toán.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Từ bằng chứng tại evidence/gate.json và evidence/fix.json:
1. PercMLP bị CHẶN (BLOCK) bởi 5/7 quy tắc:
   - R2 (Va chạm trên log): 3.0% (upper CI95 3.7%) vs baseline 0.0% + 1.0% -> FAIL.
   - R3 (Lát cắt phanh): 10.0% (upper CI95 15.1%) vs baseline 0.0% + 5.0% -> FAIL.
   - R4 (Lưới cắt đầu): diện tích lỗi 23.6% vs baseline 0.0% + 2.0% -> FAIL.
   - R5 (Ổn định giữa các seed): dao động 0.7% .. 3.0% (chênh lệch 2.3% > 1.0%) -> FAIL.
   - R6 (Thế giới phản ứng): 33.8% lưới vs baseline 10.2% + 2.0% -> FAIL.
2. Hai nguyên nhân gốc và phương pháp sửa:
   - Nguyên nhân 1 (R4, R6): Dữ liệu log tự nhiên thiếu các ca xe khác cắt đầu ở cự ly gần. Khắc phục: sinh 197050 cửa sổ từ 20000 kịch bản cắt đầu mô phỏng (cutin_windows) với nhãn lái chuẩn của IDM.
   - Nguyên nhân 2 (R2, R3, R5): Lệch phân phối (covariate shift) do sai số tích lũy trong vòng kín. Khắc phục: áp dụng thuật toán DAgger qua 2 vòng lặp (sinh thêm hơn 700000 cửa sổ trạng thái mà mạng tự lái gặp phải và mời chuyên gia IDM gán nhãn).
3. Kết quả mô hình PercMLP+: Sau khi sửa, PercMLP+ vượt qua cả 7 quy tắc và đạt quyết định PHÁT HÀNH (RELEASE): va chạm vòng kín giảm xuống 0.2..0.3% (upper CI 0.5% qua R2), lát cắt phanh giảm còn 0.5% (upper 2.9% qua R3), lưới cắt đầu giảm xuống 1.0% (qua R4), độ ổn định seed đạt chênh lệch 0.1% (qua R5), thế giới phản ứng đạt 11.3% (qua R6).
4. Đánh giá phát hành: PercMLP+ không tốt hơn IDM vì toàn bộ nhãn sửa lỗi đều học từ IDM, bản chất nó chỉ là một bộ xấp xỉ hàm nơ-ron học theo chuyên gia IDM. Trên kịch bản cruise mới 20 m/s (chưa từng thấy khi train <= 16 m/s), IDM đạt 0.0% lỗi trong khi PercMLP+ vẫn dính 0.3..0.7% lỗi. Chưa nên phát hành ngay ra xe thật; tôi đề xuất cần kiểm thử thêm trong simulator 3D hoàn chỉnh với độ trễ cảm biến, nhiễu thời tiết và trên sa hình thực nghiệm khép kín trước khi đưa lên xe thương mại.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Mặc dù PercMLP+ đã vượt qua cả 7 quy tắc an toàn của cổng kiểm định, bài lab này vẫn tồn tại 3 giới hạn cốt tử:
1. Mô hình hóa động học 1 chiều (1-D Longitudinal Only): Mô phỏng giả định mọi xe di chuyển trên một đường thẳng hoàn hảo, bỏ qua chuyển động ngang (lateral), góc đánh lái, trượt bánh, giới hạn ma sát lốp khi phanh trên đường ướt/băng tuyết, và khả năng đánh lái né chướng ngại vật thay vì chỉ đạp phanh đứng xe. Trong thực tế, một bộ điều khiển chỉ biết phanh mà không biết kết hợp chuyển làn né tránh sẽ gây nguy hiểm hoặc tắc nghẽn giao thông.
2. Giả định nhận thức hoàn hảo và không trễ: Bộ lập kế hoạch nhận được giá trị khoảng cách (gap) và vận tốc tương đối chính xác tuyệt đối ở tần số 10 Hz (bước thời gian dt = 0.1s). Trong xe thật, hệ thống camera/radar/LiDAR chịu ảnh hưởng nặng nề bởi thời tiết xấu (mưa, sương mù, ngược sáng), trễ đường truyền bus CAN (100 - 300 ms), lỗi nhận diện sai (false positive / false negative) và hiện tượng mất bám vết (tracking loss) vật thể.
3. Sự đơn giản hóa hành vi của các tác nhân xung quanh: Xe dẫn chỉ phát lại thụ động từ log quá khứ và xe sau được mô hình hóa bằng một phương trình IDM lý tưởng có trễ 0.6s. Thế giới thực phức tạp hơn rất nhiều với hành vi đa dạng và khó đoán của tài xế con người, xe máy luồn lách, người đi bộ băng cắt bất ngờ và tương tác xã hội (negotiation) tại nút giao.

Hành động tiếp theo để thu hẹp khoảng cách thực tế:
- Tích hợp mô hình AI vào các bộ mô phỏng 3D độ chân thực cao (như CARLA, nuPlan, CARLA-Autoware) với mô hình động lực học xe 6 bậc tự do (6-DoF) và cảm biến mô phỏng chùm tia LiDAR/ảnh camera thực tế.
- Kiểm thử trên hệ thống Hardware-in-the-Loop (HIL) để đo lường chính xác độ trễ thực thi trên chip nhúng chuyên dụng (NVIDIA Orin / Drive).
- Tiến hành thử nghiệm thực địa trên đường thử sa hình biệt lập (proving ground) trước khi xem xét phát hành tính năng hỗ trợ lái trên xe thương mại.
