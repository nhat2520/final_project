# Final Project - Analysis of the Facebook fanpage

This is the final project of the course "Data Processing with Python".
- The project uses the facebook_scraper library to crawl data from the GOAL football fanpage. After that, the matplotlib library is used to analyze interactions and posts, from which suggestions are made for how the fanpage can improve.
- Please refer to the following data analysis report:

# Báo cáo phân tích dữ liệu Fanpage Facebook

## 1. Thu nhập dữ liệu

- **Import các thư viện cần dùng**
    
    ```jsx
    from facebook_scraper import get_posts
    import facebook_scraper as fs
    import pandas as pd
    import numpy as np
    import time
    ```
    
- **Tạo các biến hằng cần thiết**
    
    ```jsx
    FANPAGE_LINK ="GOAL"
    FOLDER_PATH = "Raw Data/"
    
    PAGES_NUMBER = 74
    ```
    
- **Tiến hành crawl khi đã có sẵn id và mật khẩu của tài khoản**
    
    ```jsx
    count = 0
    post_list = []
    for post in get_posts(
          FANPAGE_LINK,
          options={"comments":True,  "reactions": True, "allow_extra_requests": True, "posts_per_page": 10},
          extra_info = True,
          pages = PAGES_NUMBER,
          credentials=("100094593036597", "GFTm3190")
        ):
        if count > 100:
          time.sleep(3600)
          count = 0
        post_list.append(post)
        count += 1
    ```
    
- **Chuyển đổi thành file csv**
    
    ```jsx
    post_df_full = pd.DataFrame(columns=post_list[0].keys(), index=range(len(post_list)), data=post_list)
    
    # To df
    path=FOLDER_PATH + FANPAGE_LINK + ".csv"
    post_df_full.to_csv(path, index=False)
    print(path)
    ```
    

## 2. Làm sạch và sắp xếp dữ liệu

### **Bước 1:**  Xóa những cột không chứa dữ liệu (nan)

Trong quá trình crawl data, các trường dữ liệu như ảnh video có thể sẽ không xuất hiện vì bài viết là thuần text ⇒ drop

```jsx
raw_df = pd.read_csv('Raw Data/GOAL.csv')
raw_df = raw_df.dropna(axis=1, how='all')
```

### **Bước 2:**  Loại bỏ trùng lặp dữ liệu

Sẽ có rất nhiều bài viết sẽ bị lặp lại do lỗi trong quá trình thu thập dữ liệu hoặc do nghẽn mạng nên cần loại bỏ các bài viết giống nhau thông qua post_id

```jsx
raw_df = raw_df.drop_duplicates(subset='post_id')
```

### **Bước 3:**  Ta cần tạo 2 bảng với các cột hay các giá trị được chọn lọc và cần thiết cho việc phân tích

- Bảng 1: chứa các trường post text, date, day, posting time, reactions count, like, love, haha, wow, sad, care, comments, shares
    
    ```jsx
    table1_columns = ['text', 'comments', 'shares', 'reaction_count']
    
    table1_df = raw_df[table1_columns].copy()
    
    # Tạo các cột mới từ reactions
    table1_df['like'] = raw_df['new_reactions'].apply(lambda x: x.get('like', 0))
    table1_df['love'] = raw_df['new_reactions'].apply(lambda x: x.get('love', 0))  
    table1_df['haha'] = raw_df['new_reactions'].apply(lambda x: x.get('haha', 0))
    table1_df['wow'] = raw_df['new_reactions'].apply(lambda x: x.get('wow', 0))
    table1_df['sad'] = raw_df['new_reactions'].apply(lambda x: x.get('sad', 0))  
    table1_df['angry'] = raw_df['new_reactions'].apply(lambda x: x.get('angry', 0))
    
    #Tạo cột time
    raw_df['time'] = pd.to_datetime(raw_df['time'])
    
    #Chia cột time
    table1_df['date'] = raw_df['time'].dt.date
    table1_df['day'] = raw_df['time'].dt.day_name()
    table1_df['posting_time'] = raw_df['time'].dt.hour
    
    # Hiển thị DataFrame mới
    table1_df = pd.DataFrame(table1_df)
    table1_df
    ```
    
- Bảng 2: chứa các trường comment, date, day, time
    
    Để tách được trường comment từ trường comments_full của file raw ban đầu chúng ta cần dựa vào các key word để lấy text và ngày giờ comments
    
    ```jsx
    import re
    from datetime import datetime
    import pandas as pd
    
    comments = []
    date = []
    day = []
    post_time = []
    
    for i in range(len(raw_df)):
        s = raw_df.iloc[i]['comments_full'].split("{")
        for c in s:
            if 'comment_text' in c:
                comment = re.search(r"'comment_text': \"(.+?)\", 'comment_time'", c)
                try:
                    if comment:
                        comments.append(comment.group(1))
                    else:
                        comment = re.search(r"'comment_text': \'(.+?)\', 'comment_time'", c)
                        comments.append(comment.group(1))
                    
                    t = re.search(r"'comment_time': (.+?)'comment_image", c).group(1)
                    dtime = datetime.strptime(t, "datetime.datetime(%Y, %m, %d, %H, %M), ")
                    dtime = pd.to_datetime(dtime)
                    date.append(dtime.date())
                    day.append(dtime.day_name())
                    post_time.append(dtime.time())
                except AttributeError as e:
                    print(f"Error processing comment: {e}")
    ```
    

## 3.Phân tích dữ liệu

### Thống kê cơ bản

1. **Tổng số bài đăng: (**22/9/2023 - 24/11/2023) là 743 bài ⇒ trung bình 1 ngày page đăng tổng cộng 12 bài viết
2. **Tổng lượt tương tác  :** với 11 triệu lượt tương tác, 350k lượt comments và  gần 150k lượt share là một con số khá tốt đối với 1 fanpage có tới 19 triệu lượt người theo dõi trong 2 tháng vừa qua

### Thống kê chi tiết

1. **Sự tương quan giữa các loại cảm xúc:**
    
    **Biểu đồ 1**
    
    ![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled.png)
    

Từ biểu đồ trên ta thấy:

- Lượng like và love chiếm số lượng tương tác lớn lượt tương tác cảm xúc chứng tỏ rằng những bài đăng nhận được sự ủng hộ và yêu mến của cộng đồng.
- Lượng haha cũng chiếm số lượng đáng kể nhưng không quá nhiều ⇒ page cũng có đăng những bài viết mang tính hài hước và giải trí để tăng tương tác và giữ chú ý của khán giả.
- Lượng angry chiếm số lượng rất rất nhỏ không đáng kể ⇒ page không đăng những bài viết có ý kiến tiêu cực , đây là điểm rất tốt.
- Cảm xúc wow cũng chiếm 1 phần nhỏ trong biểu đồ ⇒ các bài viết không có tính bất ngờ, giật gân

Tổng kết lại thì đây là một trang tin tức uy tín, họ đăng rất ít thông tin giật gân, chủ yếu là cập nhật tin tức và mang tính tính cực 

**Biểu đồ 2**

![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%201.png)

Để tìm hiểu rõ hơn về sự tương quan giữa các tương tác với nhau ta có thể tham khảo biểu đồ trên

1. **Tương tác trung bình của 1 bài đăng:**
    
    ![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%202.png)
    
    Lượng tương tác của page dao động trong khoảng 10-20 người tương tác trên 1 bài viết . Trung bình (mean) của các bài đăng trên page là 15 nghìn lượt. Có thể thấy đây là 1 con số tương tác khá tốt so với lượng theo dõi của 1 page có 19 triệu follower
    
2. **Top 5 bài viết có lượng reaction và comment lớn nhất**
    - **Top 5 comments**
        
        ![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%203.png)
        
        Các bài viết đều thu hút lượng tranh luận rất lớn. Trong đó 2 bài post đầu tiên đều liên quan đến 2 nhân vật nổi tiếng của bóng đá hiện đại đó là Messi và Ronaldo. 
        
    - Top 5 reactions:
        
        ![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%204.png)
        
        Tuy bài viết đầu có lương comments không quá lớn nhưng lại nhận được sự tương tác khủng lên tới gần 500k reactions. Tựu chung lại 5 bài post trên đều có nội dung liên quan tới các world class của thế giới rời bỏ các giải đấu chuyên nghiệp để tham gia giải vô địch quốc gia của Ả Rập
        
3. **Lượng bài đăng bài viết theo giờ và theo ngày**
    - **Theo giờ**
        
        ![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%205.png)
        
        Ta thấy các bài đăng được đăng nhiều nhất tại vào khoảng 16-19 giờ chiều, tuy nhiên đây là page có trụ sở bên nước anh nên hầu hết lượng người theo dõi có múi giờ lệch so với Việt Nam (Chênh 7 tiếng) vậy nên giá trị đó sẽ nằm trong khoảng 9-12 giờ sáng 
        
    - **Theo ngày**
        
        ![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%206.png)
        
        Nhìn chung các ngày trong tuần, page đăng với số lượng khá đồng đều. Tuy nhiên có chủ nhật và thứ 2 chênh hơn các ngày khác 1 chút là vì đó là 1 ngày sau khi các giải đấu bóng đá Châu Âu diễn ra
        
    
4. **Số lượng tương tác nhận được của bài viết khi đăng theo giờ và theo ngày**
    - **Theo giờ**
        
        ![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%207.png)
        
        Ta thấy các trường tương tác điều có giá trị lớn nhất tại 4 giờ sáng, tuy nhiên đây là page có trụ sở bên nước anh nên hầu hết lượng người theo dõi có múi giờ lệch so với Việt Nam (Chênh 7 tiếng) vậy nên giá trị lớn nhất đó chính là 9 giờ tối. Đây là khung giờ rất lý tưởng để mọi người có thể tiếp cận và tương tác
        
    - **Theo ngày:**
        
        ![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%208.png)
        
    
    Ta thấy lượng tương tác, comment, shares đều đạt đỉnh tại thứ 3, ngày mà lượng người tiếp cận được với những bài viết mà page đã đăng vào thứ 2 (thứ 2 là ngày mà page đăng nhiều post nhất như đã phân tích ở trên) lên tối đa 
    

6. **Những từ ngữ được nhắc đến nhiều nhất**

**Biểu đồ 1: từ được nhắc nhiều trong comment**

![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%209.png)

**Biểu đồ 2: biểu đồ Worldcloud (những từ được nhắc đến trong bài viết)**

![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%2010.png)

Các cầu thủ được nhắc đến nhiều nhất đó chính là:

- Messi
- Ronaldo
- Bellingham
- Haaland
- Mbappe

Điều này không khó hiểu khi mà 2 cầu thủ đầu chính là những vĩ nhân của bóng đá hiện đại ngày nay (Tuy nhiên ronaldo vẫn có phần kém nổi bật hơn khi phông chữ của Messi đậm hơn). Bên cạnh đó là 3 sao trẻ đang thi đấu rất xuất sắc và được mong chờ để làm messi và ronaldo thứ 2 trong tương lai

Những câu lạc bộ được nhắc đến nhiều nhất:

- Manchester United
- Manchester City
- Chelsea
- Real Marrid
- Bacerlona

MU là clb bòng đá có lượng fan hâm mộ đông đảo nhất trên thế giới (không biết tại sao) sau đó là các clb hàng đầu của giải Ngoại Hạng Anh và Tây Ban Nha

1. **Thời gian mọi người hay comment nhất:**
    
    ![Untitled](Ba%CC%81o%20ca%CC%81o%20pha%CC%82n%20ti%CC%81ch%20du%CC%9B%CC%83%20lie%CC%A3%CC%82u%20Fanpage%20Facebook%205e2bcc0030de4131a0b581d5b055e42e/Untitled%2011.png)
    
    Có thể thấy mọi người hay comment vào cuối tuần và t2 đầu tuần ⇒ đó là khoảng thời gian sau khi các page đăng bài về các cầu thủ và các giải bóng đá châu Âu nhiều nhất
