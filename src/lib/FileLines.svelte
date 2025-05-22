<script>
    import * as d3 from 'd3';
  
    /* ───────── props ───────── */
    export let lines  = [];    // [{ file : 'foo.txt', … }, …]
    export let width  = 1200;  // overall SVG width (parent can override)
  
    /* ───────── layout constants ───────── */
    const linesPerDot    = 10;   // ⇢ one "•" represents N source lines
    const rowHeight      = 24;   // space for the filename + line-count
    const dotsColumnX    = 260;  // where the dot grid starts (px)
    const approxDotWidth = 6;    // empirical width of a dot glyph (px)
    const dotRowHeight   = 12;   // vertical spacing between dot rows
  
    /* ───────── derived data ───────── */
    // files = [{ name : 'foo.txt', lines : [...] }, …]
    $: files = d3.groups(lines, d => d.file)
                 .map(([name, group]) => ({ name, lines: group }));
  
    /* add each file’s visual height (header + dot rows) */
    $: filesWithHeights = files.map(f => {
          const totalDots      = Math.ceil(f.lines.length / linesPerDot);
          const maxDotsPerRow  = Math.floor((width - dotsColumnX) / approxDotWidth) || totalDots;
          const dotRows        = Math.ceil(totalDots / maxDotsPerRow);
          return { ...f, groupHeight: rowHeight + dotRows * dotRowHeight };
        });
  
    /* cumulative y positions for the groups */
    $: positions = filesWithHeights.reduce((acc, f, i) => {
          acc.push(i === 0 ? 0 : acc[i-1] + filesWithHeights[i-1].groupHeight);
          return acc;
        }, []);
  
    /* helper: build <tspan> bunch with “•” rows */
    function generateDots(file) {
      const totalDots     = Math.ceil(file.lines.length / linesPerDot);
      const maxDotsPerRow = Math.floor((width - dotsColumnX) / approxDotWidth) || totalDots;
      const rows          = Math.ceil(totalDots / maxDotsPerRow);
  
      let tspans = '';
      for (let r = 0; r < rows; ++r) {
        const count   = Math.min(maxDotsPerRow, totalDots - r * maxDotsPerRow);
        const dots    = '•'.repeat(count);
        const dy      = r === 0 ? 0 : dotRowHeight;
        tspans += `<tspan x="${dotsColumnX}" dy="${dy}">${dots}</tspan>`;
      }
      return tspans;
    }
  
    /* ───────── drawing / updating ───────── */
    let svg;          // bound in the markup
  
    $: if (svg) {
          const totalHeight = filesWithHeights.reduce((s, f) => s + f.groupHeight, 0);
  
          const svgSel = d3.select(svg)
                           .attr('width',  width)
                           .attr('height', totalHeight)
                           .style('overflow', 'hidden');
  
          /* DATA JOIN ---------------------------------------------------- */
          const groups = svgSel.selectAll('g.file')
                               .data(filesWithHeights, d => d.name);
  
          /* EXIT --------------------------------------------------------- */
          groups.exit().remove();
  
          /* ENTER -------------------------------------------------------- */
          const enter = groups.enter()
                              .append('g')
                              .attr('class', 'file');
  
          /* filename */
          enter.append('text')
               .attr('class', 'filename')
               .attr('x', 10)
               .attr('y', 0)
               .attr('dominant-baseline', 'hanging');
  
          /* line-count */
          enter.append('text')
               .attr('class', 'linecount')
               .attr('x', 10)
               .attr('y', 12)
               .attr('dominant-baseline', 'hanging');
  
          /* dot grid */
          enter.append('text')
               .attr('class', 'unit-dots')
               .attr('x', dotsColumnX)
               .attr('y', rowHeight)          // first dot row sits below the header
               .attr('fill', '#1f77b4')
               .attr('dominant-baseline', 'hanging');
  
          /* ENTER + UPDATE MERGE ---------------------------------------- */
          const merged = enter.merge(groups);
  
          merged.attr('transform', (d, i) => `translate(0, ${positions[i]})`);
  
          merged.select('.filename').text(d => d.name);
          merged.select('.linecount').text(d => `${d.lines.length} lines`);
          merged.select('.unit-dots').html(generateDots);
    }
  </script>
  
  <svg bind:this={svg}></svg>
          