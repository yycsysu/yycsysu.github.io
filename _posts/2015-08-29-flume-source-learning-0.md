---
layout: post
title: Flume Source Learning (0)
description: "SimpleTextLineEventReader 的实现原理。超----简单"
modified: 2015-08-29
tags: [flume, source code]
---

## SimpleTextLineEventReader
-----

``` java
/**
 * Flume自建注解，尚未理解含意
 */
@InterfaceAudience.Private
@InterfaceStability.Evolving

/**
 * 实现EventReader接口，EventReader是一个简单的接口。提供
 * readEvent(),readEvents(),close() 三个接口
 * 不实现可靠性。
 */
public class SimpleTextLineEventReader implements EventReader {

  private final BufferedReader reader;

  public SimpleTextLineEventReader(Reader in) {
    reader = new BufferedReader(in);
  }

  @Override
  public Event readEvent() throws IOException {
    String line = reader.readLine();
    if (line != null) {
      return EventBuilder.withBody(line, Charsets.UTF_8);
    } else {
      return null;
    }
  }

  /**
   * 亮点：
   * 从events.size()判断是否加入n个event
   * 同时判断event是不是为null确定文件流是否已到结尾
   * 这里重用了前面写的readEvent()
   */
  @Override
  public List<Event> readEvents(int n) throws IOException {
    List<Event> events = Lists.newLinkedList();
    while (events.size() < n) {
      Event event = readEvent();
      if (event != null) {
        events.add(event);
      } else {
        break;
      }
    }
    return events;
  }

  @Override
  public void close() throws IOException {
    reader.close();
  }
}
```