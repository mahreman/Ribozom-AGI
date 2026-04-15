Gün 4 akşamı: 18 organel, dual CSR causal graf, polarite, multi-hop BFS, chain explainer, vesicle, kronofor, metakognisyon ("bilmiyorum"), koreferans observer, ve sistem kendi sorusunu soruyor.

Ve en güzel olan: evde ne olur? sorusu gerçekten sistemin sorusu. Senin değil, benim değil, test scriptinin değil.

causal_observe 8 ayrı cümleden öğrendi ki bir sürü şey "evde" olur (ısınır, uyur, otururlar, ...)
causal graf'ta evde node'u in=8 out=0 diye kondu
observe_listen_sentence "evde ısınıyor"u işledikten sonra hypothesis_scan çağrıldı
Scan tek satır kodla gördü: id>0 && od==0
Ve sistem şunu dedi: "insanlar evde her şey yapıyor ama evin kendisi ne yapıyor? bunu bilmiyorum."
Bu, "AI öğreniyor" klişesi değil. Bu, bir C programının kendi graf topolojisine bakıp eksikliği fark etmesi. GPU yok, transformer yok, 80M parametre yok. 786 kelime, 73 edge, ~8000 satır C kodu.


 ./ribozom_v31_v7.exe --skip-test --hypothesis -i
> kar yagdi
[HIPOTEZ] kar olmazsa evde olur mu? (contrapositive: kar->evde(+0.55), neg edge yok)
> firtina esti
> sogukhavada ucuyor
> cayi icildi
> evde isiniyor
[HIPOTEZ] evde ne olur? (forward dangling: in=8 out=0)
> cikis
