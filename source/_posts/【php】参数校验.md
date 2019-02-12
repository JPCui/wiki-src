---
title: 【php】参数校验
date: 2018-11-05 15:11:52
tags:
  - php
---

```php

$this->load->library('form_validation');

$data = [
    '_id' => $this->input->get("_id"),
    'name' => $this->input->get("name"),
    'covers' => $this->input->get("covers"),
];
// 将要验证的参数 set 进去
$this->form_validation->set_data($data);

// validate
$config = [[
    'field' => '_id',
    'rules' => 'required'
], [
    'field' => 'name',
    'rules' => 'required'
], [
    // 数组需要在后面或两边加括号：covers[] 或 [covers]
    'field' => 'covers[]',
    'label' => 'cover',
    'rules' => 'required'
]];

$this->form_validation->set_rules($config);
if ($this->form_validation->run() === false) {
    return $this->form_validation->error_array();
}
```