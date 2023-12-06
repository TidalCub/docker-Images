FROM python:3.10.12

WORKDIR ./

RUN git clone https://github.com/TidalCub/StormsBurrow

RUN pip3 install Flask
RUN pip3 install gunicorn

EXPOSE 8000

CMD ["gunicorn", "-b", "0.0.0.0:8000", "StormsBurrow:app"]