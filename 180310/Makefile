INPUT	= input.md
OUTPUT	= slide.html
PANDOC	= pandoc
REVEAL	= reveal.js

all:	$(OUTPUT)

$(OUTPUT)	:	$(INPUT)	$(REVEAL)
				$(PANDOC) $(INPUT) -s -t revealjs -o $(OUTPUT)

$(REVEAL)	:	
				ln -s ../$(REVEAL) .

clean:
				rm -f $(REVEAL) $(OUTPUT)
