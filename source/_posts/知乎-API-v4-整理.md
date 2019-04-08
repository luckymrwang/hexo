title: 知乎 API v4 整理
date: 2019-04-06 21:21:53
tags: [Api]
---

### 问题的答案

 - URL: `https://www.zhihu.com/api/v4/questions/{qid}/answers`
 - 参数：
 	- limit：数量，最大 20
 	- offset：起始位置，从零开始
 	- sort_by：{default, created}，表示默认排序或者时间排序
 	- include：额外信息，包括
`is_normal,is_sticky,collapsed_by,suggest_edit,comment_count,collapsed_counts,reviewing_comments_count,can_comment,content,editable_content,voteup_count,reshipment_settings,comment_permission,mark_infos,created_time,updated_time,relationship.is_author,voting,is_thanked,is_nothelp,upvoted_followees;author.is_blocking,is_blocked,is_followed,voteup_count,message_thread_token,badge[?(type=best_answerer)].topics`

<!-- more -->
- eg: [https://www.zhihu.com/api/v4/questions/51196509/answers?include=data[\*].is_normal,admin_closed_comment,reward_info,is_collapsed,annotation_action,annotation_detail,collapse_reason,is_sticky,collapsed_by,suggest_edit,comment_count,can_comment,content,editable_content,voteup_count,reshipment_settings,comment_permission,created_time,updated_time,review_info,relevant_info,question.detail,excerpt,relationship.is_authorized,is_author,voting,is_thanked,is_nothelp,is_labeled,is_recognized;data[*].mark_infos[*].url;data[*].author.follower_count,badge[*].topics&limit=5&offset=10&platform=desktop&sort_by=default](https://www.zhihu.com/api/v4/questions/51196509/answers?include=data[*].is_normal,admin_closed_comment,reward_info,is_collapsed,annotation_action,annotation_detail,collapse_reason,is_sticky,collapsed_by,suggest_edit,comment_count,can_comment,content,editable_content,voteup_count,reshipment_settings,comment_permission,created_time,updated_time,review_info,relevant_info,question.detail,excerpt,relationship.is_authorized,is_author,voting,is_thanked,is_nothelp,is_labeled,is_recognized;data[*].mark_infos[*].url;data[*].author.follower_count,badge[*].topics&limit=5&offset=10&platform=desktop&sort_by=default)

### 指定作者的回答

- URL：`https://www.zhihu.com/api/v4/members/{un}/answers`

- 参数：
	- limit：数量，最大 20
	- offset：起始位置，从零开始
	- sort_by：{default, created}，表示默认排序或者时间排序
	- include：额外信息，包括`is_normal,suggest_edit,comment_count,collapsed_counts,reviewing_comments_count,can_comment,content,voteup_count,reshipment_settings,comment_permission,mark_infos,created_time,updated_time,relationship.voting,is_author,is_thanked,is_nothelp,upvoted_followees;author.badge[?(type=best_answerer)].topics`

- eg: [https://www.zhihu.com/api/v4/members/xxx/answers?offset=5&limit=15&sort_by=created&&include=data[\*].is_normal,suggest_edit,comment_count,collapsed_counts,reviewing_comments_count,can_comment,content,voteup_count,reshipment_settings,comment_permission,mark_infos,created_time,updated_time,relationship.voting,is_author,is_thanked,is_nothelp,upvoted_followees;data[*].author.badge[?(type=best_answerer)].topics](https://www.zhihu.com/api/v4/members/ying-ye-78/answers?offset=5&limit=15&sort_by=created&&include=data[*].is_normal,suggest_edit,comment_count,collapsed_counts,reviewing_comments_count,can_comment,content,voteup_count,reshipment_settings,comment_permission,mark_infos,created_time,updated_time,relationship.voting,is_author,is_thanked,is_nothelp,upvoted_followees;data[\*].author.badge[?(type=best_answerer)].topics)


### 自身信息

- URL：`https://www.zhihu.com/api/v4/me`
- 参数：
	- include：额外信息，包括`locations,employments,gender,educations,business,voteup_count,thanked_Count,follower_count,following_count,cover_url,following_topic_count,following_question_count,following_favlists_count,following_columns_count,avatar_hue,answer_count,articles_count,pins_count,question_count,columns_count,commercial_question_count,favorite_count,favorited_count,logs_count,included_answers_count,included_articles_count,included_text,message_thread_token,account_status,is_active,is_bind_phone,is_force_renamed,is_bind_sina,is_privacy_protected,sina_weibo_url,sina_weibo_name,show_sina_weibo,is_blocking,is_blocked,is_following,is_followed,is_org_createpin_white_user,mutual_followees_count,vote_to_count,vote_from_count,thank_to_count,thank_from_count,thanked_count,description,hosted_live_count,participated_live_count,allow_message,industry_category,org_name,org_homepage,badge[?(type=best_answerer)].topics`

### 关注者信息
- URL：`https://www.zhihu.com/api/v4/members/{un}/followees`
- 参数：
	- limit：数量，最大 20
	- offset：起始位置，从零开始
	- include：额外信息，包括`answer_count,articles_count,gender,follower_count,is_followed,is_following,badge[?(type=best_answerer)].topics`

- eg: [https://www.zhihu.com/api/v4/members/xxx/followees?include=data[\*].answer_count,articles_count,gender,follower_count,is_followed,is_following,badge[?(type=best_answerer)].topics&offset=0&limit=20](https://www.zhihu.com/api/v4/members/yksin/followees?include=data[*].answer_count,articles_count,gender,follower_count,is_followed,is_following,badge[?(type=best_answerer)].topics&offset=0&limit=20)

### 粉丝信息
- URL：https://www.zhihu.com/api/v4/members/{un}/followers
- 参数：
	- limit：数量，最大 20
	- offset：起始位置，从零开始
	- include：额外信息，包括`answer_count,articles_count,gender,follower_count,is_followed,is_following,badge[?(type=best_answerer)].topics`

- eg: [https://www.zhihu.com/api/v4/members/xxx/followers?include=data[\*].answer_count,articles_count,gender,follower_count,is_followed,is_following,badge[?(type=best_answerer)].topics&offset=0&limit=20](https://www.zhihu.com/api/v4/members/yksin/followers?include=data[*].answer_count,articles_count,gender,follower_count,is_followed,is_following,badge[?(type=best_answerer)].topics&offset=0&limit=20)


### 收藏夹信息

- URL：`https://api.zhihu.com/collections/{cid}`

### 收藏夹中的回答摘要

- URL：`https://api.zhihu.com/collections/{cid}/answers`
- 参数：
	- limit：数量，最大 20
	- offset：起始位置，从零开始

	
### 知乎话题

- URL：`https://www.zhihu.com/api/v4/topics/{tid}`

### 知乎话题精华

- URL：`https://www.zhihu.com/api/v4/topics/{tid}/feeds/essence`
- 参数：
	- limit：数量，最大 20
	- offset：起始位置，从零开始
	- include：额外信息，包括
`data[?(target.type=topic_sticky_module)].target.data[?(target.type=answer)].target.content,relationship.is_authorized,is_author,voting,is_thanked,is_nothelp;data[?(target.type=topic_sticky_module)].target.data[?(target.type=answer)].target.is_normal,comment_count,voteup_count,content,relevant_info,excerpt.author.badge[?(type=best_answerer)].topics;data[?(target.type=topic_sticky_module)].target.data[?(target.type=article)].target.content,voteup_count,comment_count,voting,author.badge[?(type=best_answerer)].topics;data[?(target.type=topic_sticky_module)].target.data[?(target.type=people)].target.answer_count,articles_count,gender,follower_count,is_followed,is_following,badge[?(type=best_answerer)].topics;data[?(target.type=answer)].target.annotation_detail,content,hermes_label,is_labeled,relationship.is_authorized,is_author,voting,is_thanked,is_nothelp;data[?(target.type=answer)].target.author.badge[?(type=best_answerer)].topics;data[?(target.type=article)].target.annotation_detail,content,hermes_label,is_labeled,author.badge[?(type=best_answerer)].topics;data[?(target.type=question)].target.annotation_detail,comment_count`

- eg: [https://www.zhihu.com/api/v4/topics/19555634/feeds/essence?include=data[?(target.type=topic_sticky_module)].target.data[?(target.type=answer)].target.content,relationship.is_authorized,is_author,voting,is_thanked,is_nothelp;data[?(target.type=topic_sticky_module)].target.data[?(target.type=answer)].target.is_normal,comment_count,voteup_count,content,relevant_info,excerpt.author.badge[?(type=best_answerer)].topics;data[?(target.type=topic_sticky_module)].target.data[?(target.type=article)].target.content,voteup_count,comment_count,voting,author.badge[?(type=best_answerer)].topics;data[?(target.type=topic_sticky_module)].target.data[?(target.type=people)].target.answer_count,articles_count,gender,follower_count,is_followed,is_following,badge[?(type=best_answerer)].topics;data[?(target.type=answer)].target.annotation_detail,content,hermes_label,is_labeled,relationship.is_authorized,is_author,voting,is_thanked,is_nothelp;data[?(target.type=answer)].target.author.badge[?(type=best_answerer)].topics;data[?(target.type=article)].target.annotation_detail,content,hermes_label,is_labeled,author.badge[?(type=best_answerer)].topics;data[?(target.type=question)].target.annotation_detail,comment_count;&limit=10&offset=10](https://www.zhihu.com/api/v4/topics/19555634/feeds/essence?include=data[?(target.type=topic_sticky_module)].target.data[?(target.type=answer)].target.content,relationship.is_authorized,is_author,voting,is_thanked,is_nothelp;data[?(target.type=topic_sticky_module)].target.data[?(target.type=answer)].target.is_normal,comment_count,voteup_count,content,relevant_info,excerpt.author.badge[?(type=best_answerer)].topics;data[?(target.type=topic_sticky_module)].target.data[?(target.type=article)].target.content,voteup_count,comment_count,voting,author.badge[?(type=best_answerer)].topics;data[?(target.type=topic_sticky_module)].target.data[?(target.type=people)].target.answer_count,articles_count,gender,follower_count,is_followed,is_following,badge[?(type=best_answerer)].topics;data[?(target.type=answer)].target.annotation_detail,content,hermes_label,is_labeled,relationship.is_authorized,is_author,voting,is_thanked,is_nothelp;data[?(target.type=answer)].target.author.badge[?(type=best_answerer)].topics;data[?(target.type=article)].target.annotation_detail,content,hermes_label,is_labeled,author.badge[?(type=best_answerer)].topics;data[?(target.type=question)].target.annotation_detail,comment_count;&limit=10&offset=10)
