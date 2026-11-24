# docker build -t todo . && docker run --rm -p 3000:3000 todo
FROM ortussolutions/boxlang:cli-1.19.0@sha256:8fe238b155ea3c0e8d9ff19db2f6f40f518e90bd89f5740f32aa7cbf320364ab
WORKDIR /app
RUN rm -rf /app/* /app/.[!.]*
RUN install-bx-module boxlang-express@0.2.29,boxlang-express-mvc@0.1.0,bx-esapi@1.12.0 --local
COPY server.bxs ./
COPY config/ config/
COPY handlers/ handlers/
COPY models/ models/
COPY layouts/ layouts/
COPY views/ views/
RUN useradd --system --create-home --home-dir /home/app --shell /usr/sbin/nologin app \
	&& cp -a /root/.boxlang /home/app/.boxlang \
	&& chown -R app:app /home/app /app
USER app
ENV HOME=/home/app
ENV PORT=3000
EXPOSE 3000
CMD ["boxlang", "server.bxs"]
