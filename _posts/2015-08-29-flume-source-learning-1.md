---
layout: post
title: Flume Source Learning (1)
description: "ReliableSpoolingFileEventReader.class"
modified: 2015-08-29
tags: [flume, source code]
image:
  feature: abstract-12.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

## Flume Source Learning (1)- ReliableSpoolingFileEventReader
-----

~~~ java
/**
 * Flume自建注解，尚未理解含意
 */
@Private
@Evolving

/**
 * 实现EventReader接口，EventReader是一个简单的接口。提供
 * readEvent(),readEvents(),close() 三个接口
 * 不实现可靠性。
 */
public class SimpleTextLineEventReader implements EventReader {
    private final BufferedReader reader;

    public SimpleTextLineEventReader(Reader in) {
        this.reader = new BufferedReader(in);
    }

    public Event readEvent() throws IOException {
        String line = this.reader.readLine();
        return line != null?EventBuilder.withBody(line, Charsets.UTF_8):null;
    }

    /**
     * 亮点：
     * 从events.size()判断是否加入n个event
     * 同时判断event是不是为null确定文件流是否已到结尾
     * 这里重用了前面写的readEvent()
     */
    public List<Event> readEvents(int n) throws IOException {
        LinkedList events = Lists.newLinkedList();

        while(events.size() < n) {
            Event event = this.readEvent();
            if(event == null) {
                break;
            }

            events.add(event);
        }

        return events;
    }

    public void close() throws IOException {
        this.reader.close();
    }
}
~~~


