# dosxnjos.github.io
Personal site — dosxnjos.github.io

## Design

Paleta indigo (`#6366f1` base), toggle dark/light (localStorage), Space Grotesk
(headings) + Inter (corpo), glassmorphism nos cards, tilt 3D no hover, sheen
holográfico, fade-up por `IntersectionObserver`. Rastro de cursor via
`webgl-fluid-enhanced@0.8.0` (CDN, migrado de `webgl-fluid@0.4.0` em 27/07/2026,
junto com a mesma migração em `fluid-cursor-trail`).

Config do rastro: `colorPalette: ['#4D2EFF']`, `colorful:false`,
`transparent:true`, sem sunrays, `densityDissipation:3.0`; pausa por
inatividade (3s sem mover o ponteiro) + guarda de `prefers-reduced-motion`.
`#fluid` é um `<div>`, não um `<canvas>` — o fork cria o canvas interno e
sobrescreve o `style` inline do container, por isso `position:fixed !important`
é obrigatório no CSS em vez de deixar pro elemento original.

Seção "Projetos" (lista compacta nome+descrição+link, sem thumbnail) — ao
mexer de novo, preservar a paleta indigo-first e o estilo lista, a menos que o
Gabriel peça tratamento mais pesado. `github-readme-stats` está banido (removido
09/06/2026 por instabilidade — não reintroduzir).
