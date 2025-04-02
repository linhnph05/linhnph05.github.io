---
title: Writeup Cyberjutsu Christmas OSINT 2024 challenge
published: 2025-04-02
description: 'Writeup  for Cyberjutsu Christmas OSINT 2024 challenge'
image: ''
tags: [OSINT, writeup, ctf]
category: 'writeup'
draft: false 
lang: ''
---

## Challenge
[Link](https://www.facebook.com/CyberJutsu/posts/pfbid0n8hk9AVVuP8n4P6ND5635uXt93uPbUfoyq1riwum7EHZ61Upjn9DvDPrvL5EJNaml?__cft__[0]=AZX9Cij64UsVFGcYMHHo7pQLf_OghNp98Di7cKyqbML7-DWNzp2lyX-86lloVHeFZ9jIwG4jHBKXfBQ_lTlV-jYyPZmnHqevGLzyr6Hy_3BrrRdIUqejtkPGPAV4OPJNYrkBMhmKdjuPhp45Oylv4aJNPNv9BMFWt6ewFiA_7qR8eXVmv6t089NShp-O1H5VCBI&__tn__=%2CO%2CP-R)
Nói tóm gọn là challenge yêu cầu chúng ta OSINT về một username tên là "solo_levelling", có một số dữ kiện là user đó đã xoá sạch source code và backup. Chúng ta cần tìm 3 flag có format là CBJS{...}. Oke bắt đầu thôi hehe =))

## Hướng tiếp cận

### Flag 1
Đối với bất kỳ OSINT challenge nào, đầu tiên chúng ta phải tìm tất cả thông tin, manh mối có thể có từ dữ kiện đề bài cho. Đầu tiên, ta có thể đặt một câu hỏi là: "User đó đã xoá sạch code và backup, vậy liệu ta có thể truy tìm lại code đã bị xoá không" -> Ta nghĩ ngay đến tài khoản Github, hoặc Gitlab, nơi mà lưu trữ cái repo code cũng như commit history của các repo đó -> Tìm trên github ta được tài khoản sau: https://github.com/solo-levelling21/

Ở profile đó ta thấy có đoạn mã khá là khả nghi: `
-.-. -... .--- ... ..-. .- -.- . ..--.- ..-. .-.. .- --.`, nhưng quăng lên ChatGPT decode thì ra fake flag :(( . Tiếp tục, ta vô repo project-christmas, lục lọi commit history thì thấy được đoạn comment sau:
![image](https://hackmd.io/_uploads/rJwM8bFSke.png)
Ra được flag 1: CBJS{hehehee_ban_co_de_lai_gi_trong_github_commit_hong???!}

### Flag 2
Tiếp theo, do đề chỉ cho ta mỗi username "solo_levelling" nên ta sẽ tìm tất cả thông tin liên quan đến username này. Hiện nay, có rất nhiều tool như [OSINT framework](https://osintframework.com/), chall này thì mình dùng [user-searcher](https://www.user-searcher.com/), search username thì ra link [linktree](https://linktr.ee/solo_levelling). Trong linktree thì có link đến một tài khoản [mastodon](https://mastodon.social/@solo_levelling). Trong đó lại có một link drive của một video:
![image](https://hackmd.io/_uploads/HJ6Zu-Frkx.png)
![image](https://hackmd.io/_uploads/r108_btrJg.png)

Các bạn chịu khó zoom video lên thì sẽ thấy sẽ có một đường link dẫn đến một tài khoản [Twitter](https://x.com/hackerbinhthanh), vô đó sẽ có một QR code, quét xong sẽ ra được flag 2: CBJS{tinh_mat_qua_chac_ko_sot_con_bug_nao_dau:))):D} =))

### Flag 3
Mình đã tốn thời gian khá lâu để thử lục lọi tất cả mọi thứ liên quan đến username "solo_levelling" nhưng khá là vô ích =)). Nhưng may mắn là lúc check fb coi có chall này đã có bao nhiêu solve (khả năng là những ai đã share bài public), mình sẵn tiện check cmt thì thấy có đoạn này khá sus:
![image](https://hackmd.io/_uploads/SyfQo-FB1e.png)
![image](https://hackmd.io/_uploads/rJVEiWYr1l.png)


Liên kết với dữ kiện commit history trên Github của Flag 1: 
![image](https://hackmd.io/_uploads/r1d_oWtBkg.png)

Thì có vẻ là mình đang đi đúng hướng rồi, "solo_levelling" đánh cắp code quà giáng sinh của một ông intern trên máy tính của ô đó, nguyên nhân là do ông intern cài crack của "solo_levelling"-> Quá là hợp lý, chúng ta đã tìm ra được manh mối cuối cùng rồi, phá án thôi =))

Vô [fb ông intern](https://www.facebook.com/profile.php?id=100088375063671&comment_id=Y29tbWVudDoxMjcwNDQ2NTA0MDQ4NTE3XzI5ODUwMTA4MDQ5OTU1MzE%3D), lục lọi lịch sử edit post thì ta được một [link shopee](https://shopee.vn/product/339905651/24238438584/) có phần mô tả sản phẩm quá sú:
![image](https://hackmd.io/_uploads/SksNTWKrJl.png)

Ta quăng lên hỏi ChatGPT thử thì biết đó là một dạng mật mã sử dụng khoảng trắng -> Copy đoạn đó, quăng lên mấy trang decode online là tìm được flag:
![image](https://hackmd.io/_uploads/rkFRp-Frkx.png)

Flag3: CBJS{CBJS_chuc_anh_em_giang_sinh_an_lanh!!!<33}