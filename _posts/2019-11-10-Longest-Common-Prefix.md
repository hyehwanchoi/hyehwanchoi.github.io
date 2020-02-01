---
layout: single
title:  "Longest Common Prefix"
date:   2019-11-10 21:00:00
author: HyeHwan Choi
categories: algorithm
tags:   algorithm
cover:  "/assets/instacode.png"
---

# Longest Common Prefix

Question)
Write a function to find the longest common prefix string amongst an array of strings.  
If there is no common prefix, return an empty string "".    

**Example 1:**  
Input: ["flower","flow","flight"]  
Output: "fl"    

**Example 2:**  
Input: ["dog","racecar","car"]  
Output: ""      

**Explanation: There is no common prefix among the input strings.**  

<pre><code>
public String longestCommonPrefix(String[] strs) {
  if(strs.length == 0) return ""; // 파라미터가 없을 경우 공백 반환(테스트케이스에 있었다)
  String answer = strs[0]; // 첫번째 문자열을 기준으로 만듦

  for(int i=1; i<strs.length; i++) { // 첫번째 문자열을 뺀 나머지 문자열들을 반복하여
    while(strs[i].indexOf(answer) != 0) { // 기준이 되는 문자열과 비교되는 문자열이 같지 않다면
      answer = answer.substring(0, answer.length() - 1); // 기준이 되는 문자열을 뒤에서부터 하나씩 자른다
                                                 // 같은 문자가 있을 경우 다음 문자열과 현재 남은 문자열과 비교
      if(answer.isEmpty()) return ""; // 공통된 문자열이 하나도 없을 경우 공백을 반환
    }
  }
  return answer;
}
</code></pre>

