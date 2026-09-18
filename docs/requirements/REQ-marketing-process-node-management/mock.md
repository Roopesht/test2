# Mock: Marketing Process Node Management

```text
+--------------------------------------------------------------------+
| Marketing Process: In-office batch-based courses            [+Node]|
+--------------------------------------------------------------------+
|                                                                      |
|  (1) Identify   ->  (2) Define    ->  (3) Reach     ->  (4) Capture |
|      Course           Campaign          Audience         Leads      |
|                                                              |       |
|                                                              v       |
|  (8) Follow   <-  (7) Offer    <-  (6) Measure   <-  (5) Nurture    |
|      Up               Course          Interest          Leads       |
|                                                                      |
|  [drag to reorder]  [split node]  [remove node]  [add branch]       |
+--------------------------------------------------------------------+

  Selected node: (4) Capture Leads

  +------------------------------------------------------------------+
  | Node: Capture Leads                              id: capture-leads|
  +------------------------------------------------------------------+
  | [ Level 1 ]  [ Level 2 ]  [ Automation ]  [ Tools ]  [ Notes ]    |
  +------------------------------------------------------------------+
  | Level 1 (tab selected)                                            |
  |                                                                    |
  | Description:                                                      |
  | +----------------------------------------------------------------+
  | | Capture people who respond to the campaign and create          |
  | | identifiable lead records.                                     |
  | +----------------------------------------------------------------+
  |                                                                    |
  | Input:  [ Campaign response            ]                          |
  | Output: [ Identifiable lead record      ]                         |
  |                                                                    |
  | Next node(s): [4] -> [5] Nurture Leads         [+ add next]       |
  +--------------------------------------------------------------------+

  +------------------------------------------------------------------+
  | [ Level 1 ]  [ Level 2 ]  [ Automation ]  [ Tools ]  [ Notes ]    |
  +------------------------------------------------------------------+
  | Level 2 (tab selected)                                            |
  |                                                                    |
  | Manual:                                                           |
  |  - Marketing Manager reviews new leads                            |
  |                                                                    |
  | Automated:                                                        |
  |  - Landing page collects lead information                         |
  |  - Form stores the submission                                     |
  +--------------------------------------------------------------------+

  +------------------------------------------------------------------+
  | [ Level 1 ]  [ Level 2 ]  [ Automation ]  [ Tools ]  [ Notes ]    |
  +------------------------------------------------------------------+
  | Automation (tab selected)          Status: [Partially automated v]|
  |                                                                    |
  | Trigger:    [ New lead submitted                       ]          |
  | Conditions: [ Required information is available        ]          |
  | Actions:                                                          |
  |   1. Create lead                                                  |
  |   2. Identify campaign                                            |
  |   3. Record source                                                |
  |   4. Start nurture process                                        |
  |   5. Notify responsible person                       [+ action]   |
  +--------------------------------------------------------------------+

  +------------------------------------------------------------------+
  | [ Level 1 ]  [ Level 2 ]  [ Automation ]  [ Tools ]  [ Notes ]    |
  +------------------------------------------------------------------+
  | Tools (tab selected)                                              |
  |                                                                    |
  |  - Website           https://example.com                          |
  |  - Lead System        https://example.com                [+ tool] |
  +--------------------------------------------------------------------+
```
