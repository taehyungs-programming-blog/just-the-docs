---
layout: default
title: "12. Bridge Pattern"
parent: (Desing Pattern)
grand_parent: C++
nav_order: 2
---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Bridge Pattern

🦍 구현과 추상화 개념을 분리해서 각각을 독립적으로 변형할 수 있게한다.

```cpp
#include <iostream>
using namespace std;

struct IMP3
{
    virtual void Play() = 0;
    virtual void Stop() = 0;
    virtual ~IMP3() {}
}

class IPod
{
public:
    void Play() { cout << "Play MP3" << endl; }
    void Stop() { cout << "Stop MP3" << endl; }
};

class People
{
public:
    void UseMP3(IMP3* p)
    {
        p->Play();
        p->PlayOneMinute();     // 이 함수를 추가하고 싶다면?
    }
};

int main()
{

}
```

```cpp
class MP3
{
    IMP3* pImpl;    // pimpl은 별도로 정리한 pattern참고
public:
    MP3()
    {
        pImpl = new IPod;
    }
    void Play() { pImpl->Play(); }
    void Stop() { pImpl->Stop(); }
    void PlayOneMinute() {
        // 1분 후 종료
    }
};

class People
{
public:
    void UseMP3(MP3* p)
    {
        p->Play();
        p->PlayOneMinute();     // MP3에 이 함수를 만들면된다.
        // people에서는 PlayOneMinute()이 뭔지에 관해선 알 필요가 없음
    }
};
```