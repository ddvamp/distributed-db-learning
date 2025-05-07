## Lecture 7. RAFT

Raft - алгоритм репликации лога аналогичный Multi-Paxos, но в отличие от него полностью описан и фиксирован

Поскольку здесь нет мест для обсуждения, оставлю алгоритм без внимания. К тому же, имо, он менее интересен и имеет ряд недостатков, как например откат логов и жёсткая связь safety и liveness частей

Для ознакомления можно прочесть оригинальную статью и PhD
- [Сайт про Raft](https://raft.github.io/)
- [Оригинальная статья](https://raft.github.io/raft.pdf)
- [PhD - описание полной production ready реализации](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf)

[↑ Содержание ↑](https://github.com/ddvamp/distributed-db-learning/tree/main/notes/dist-sys-mipt#содержание)\
[← Семинар 6](https://github.com/ddvamp/distributed-db-learning/blob/main/notes/dist-sys-mipt/seminars/seminar-5.md)
[Лекция 8 →](https://github.com/ddvamp/distributed-db-learning/blob/main/notes/dist-sys-mipt/lectures/lecture-8.md)
